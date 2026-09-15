# DoIP Mobile V1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a native Android APK that connects a Samsung S24 Ultra class device to a vehicle through USB-C Ethernet, discovers DoIP entities, performs ISO 13400 routing activation, sends a read-only UDS `22 F1 90` request, displays the returned VIN, and exposes a raw protocol console.

**Architecture:** Keep V1 in a single `:app` Gradle module but separate responsibilities by package. All DoIP/UDS codecs and session logic stay pure Kotlin/JVM-testable; Android-specific Ethernet selection and socket binding live behind small interfaces. The UI is Jetpack Compose and observes a single ViewModel state machine. V1 is read-only: no ClearDTC, ECU reset, SecurityAccess, RoutineControl, download/transfer, or flashing APIs are exposed.

**Tech Stack:** Kotlin 2.4.20, Android Gradle Plugin 9.3.0, Gradle 9.5.0, JDK 17, compileSdk 37, targetSdk 36, minSdk 26, Jetpack Compose BOM 2026.08.00, Activity Compose 1.13.0, Lifecycle 2.11.0, kotlinx-coroutines 1.11.0, JUnit 4.13.2.

**Spec:** `docs/superpowers/specs/2026-09-16-doip-mobile-design.md`

## Global Constraints

- Android only; package name `com.qujindai.doipmobile`.
- No CAN, CAN FD, ISO-TP, ELM327, Bluetooth, Wi-Fi VCI, proprietary VCI SDK, root, or modified Android kernel.
- Physical path is vehicle DoIP Ethernet -> USB-C Ethernet adapter/cable -> Android Ethernet transport.
- Do not bind the whole Android process to the vehicle network. Bind only DoIP sockets to the selected Ethernet `Network`.
- DoIP UDP/TCP port is `13400`.
- Default tester logical address is `0x0E80`; keep it configurable in code.
- Default routing activation type is `0x00`.
- Default DoIP protocol version for transmitted frames is `0x02`; decoder accepts a valid version/inverse-version pair and rejects malformed headers.
- V1 services are read-only: vehicle identification, routing activation, tester present only when required by an active session, and UDS ReadDataByIdentifier (`0x22`).
- V1 VIN DID is `0xF190`; a positive response must begin `62 F1 90` and yield a 17-character VIN.
- All protocol core logic must be covered by local JVM tests; no vehicle is required for CI.
- CI must run unit tests before assembling the debug APK.
- No dependency on code from repositories without a verified reusable license. Public projects such as MEET may be used as behavioral/architecture references only.

---

## File Structure

Root/build files:
- `settings.gradle.kts` — repositories and `:app` inclusion.
- `build.gradle.kts` — root Android/Kotlin/Compose plugin declarations.
- `gradle.properties` — AndroidX and JVM build flags.
- `.gitignore` — Android/Gradle generated files.
- `.github/workflows/android.yml` — JDK 17 + Gradle 9.5.0 unit-test and APK build pipeline.

Android app shell:
- `app/build.gradle.kts` — SDK levels, Compose, dependencies, test options.
- `app/src/main/AndroidManifest.xml` — network permissions and launcher activity.
- `app/src/main/java/com/qujindai/doipmobile/MainActivity.kt` — Android entrypoint only.

Protocol core:
- `app/src/main/java/com/qujindai/doipmobile/doip/DoipConstants.kt` — payload types, port, protocol constants.
- `app/src/main/java/com/qujindai/doipmobile/doip/DoipFrame.kt` — immutable decoded frame model.
- `app/src/main/java/com/qujindai/doipmobile/doip/DoipCodec.kt` — generic 8-byte DoIP header encode/decode and exact-frame reads.
- `app/src/main/java/com/qujindai/doipmobile/doip/VehicleIdentification.kt` — vehicle-identification response model/parser.
- `app/src/main/java/com/qujindai/doipmobile/doip/RoutingActivation.kt` — request builder and response parser.
- `app/src/main/java/com/qujindai/doipmobile/doip/DiagnosticMessage.kt` — `0x8001` request/response payload model/parser.
- `app/src/main/java/com/qujindai/doipmobile/uds/Uds.kt` — `22 F190`, positive response and negative response parsing.

Networking:
- `app/src/main/java/com/qujindai/doipmobile/network/SocketBinder.kt` — abstraction for binding UDP/TCP sockets to a network.
- `app/src/main/java/com/qujindai/doipmobile/network/AndroidEthernetSelector.kt` — find Ethernet `Network`, expose address/prefix, bind sockets.
- `app/src/main/java/com/qujindai/doipmobile/network/DoipDiscoveryClient.kt` — UDP `0x0001` request and `0x0004` collection.
- `app/src/main/java/com/qujindai/doipmobile/network/DoipTcpSession.kt` — TCP connect, routing activation, diagnostic request/response, close.

Application/domain:
- `app/src/main/java/com/qujindai/doipmobile/domain/DiagnosticRepository.kt` — orchestrates Ethernet -> discovery -> activation -> VIN.
- `app/src/main/java/com/qujindai/doipmobile/ui/DoipUiState.kt` — immutable screen state and log entries.
- `app/src/main/java/com/qujindai/doipmobile/ui/DoipViewModel.kt` — lifecycle state machine and coroutine ownership.
- `app/src/main/java/com/qujindai/doipmobile/ui/DoipScreen.kt` — Compose screen.

Tests:
- `app/src/test/java/com/qujindai/doipmobile/doip/DoipCodecTest.kt`
- `app/src/test/java/com/qujindai/doipmobile/doip/VehicleIdentificationTest.kt`
- `app/src/test/java/com/qujindai/doipmobile/doip/RoutingActivationTest.kt`
- `app/src/test/java/com/qujindai/doipmobile/doip/DiagnosticMessageTest.kt`
- `app/src/test/java/com/qujindai/doipmobile/uds/UdsTest.kt`
- `app/src/test/java/com/qujindai/doipmobile/network/DoipDiscoveryClientTest.kt`
- `app/src/test/java/com/qujindai/doipmobile/network/DoipTcpSessionTest.kt`
- `app/src/test/java/com/qujindai/doipmobile/domain/DiagnosticRepositoryTest.kt`

---

### Task 1: Buildable Android/Compose scaffold and CI

**Files:**
- Create: `settings.gradle.kts`
- Create: `build.gradle.kts`
- Create: `gradle.properties`
- Create: `.gitignore`
- Create: `app/build.gradle.kts`
- Create: `app/src/main/AndroidManifest.xml`
- Create: `app/src/main/java/com/qujindai/doipmobile/MainActivity.kt`
- Create: `.github/workflows/android.yml`
- Test: Gradle `testDebugUnitTest` and `assembleDebug`

**Interfaces:**
- Produces Android application id `com.qujindai.doipmobile`.
- Produces one launchable `MainActivity` and a CI artifact named `doip-mobile-debug-apk`.

- [ ] **Step 1: Add the minimum build configuration**

`build.gradle.kts`:

```kotlin
plugins {
    id("com.android.application") version "9.3.0" apply false
    id("org.jetbrains.kotlin.android") version "2.4.20" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.4.20" apply false
}
```

`settings.gradle.kts`:

```kotlin
pluginManagement {
    repositories { google(); mavenCentral(); gradlePluginPortal() }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { google(); mavenCentral() }
}
rootProject.name = "DoIP-Mobile"
include(":app")
```

`app/build.gradle.kts` must set `namespace = "com.qujindai.doipmobile"`, `compileSdk = 37`, `minSdk = 26`, `targetSdk = 36`, Java/Kotlin JVM target 17, `buildFeatures.compose = true`, and dependencies for Compose BOM `2026.08.00`, `activity-compose:1.13.0`, lifecycle `2.11.0`, coroutines `1.11.0`, and JUnit `4.13.2`.

- [ ] **Step 2: Add a compile-smoke unit test first**

Create `app/src/test/java/com/qujindai/doipmobile/BuildSmokeTest.kt`:

```kotlin
package com.qujindai.doipmobile

import org.junit.Assert.assertEquals
import org.junit.Test

class BuildSmokeTest {
    @Test fun appIdContract() {
        assertEquals("com.qujindai.doipmobile", BuildConfig.APPLICATION_ID)
    }
}
```

- [ ] **Step 3: Run the unit test and verify project compilation**

Run:

```bash
gradle --version
gradle testDebugUnitTest
```

Expected: Gradle 9.5.0/JDK 17 environment and `BuildSmokeTest` PASS.

- [ ] **Step 4: Add the launcher Activity and manifest**

Manifest permissions must include only what V1 needs:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

`MainActivity` renders a temporary `Text("DoIP Mobile")` Compose root. No protocol code belongs in the Activity.

- [ ] **Step 5: Add CI with test-before-build ordering**

Workflow steps:

```yaml
- uses: actions/checkout@v4
- uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: '17'
- uses: gradle/actions/setup-gradle@v4
  with:
    gradle-version: '9.5.0'
- run: gradle testDebugUnitTest
- run: gradle assembleDebug
- uses: actions/upload-artifact@v4
  with:
    name: doip-mobile-debug-apk
    path: app/build/outputs/apk/debug/app-debug.apk
```

- [ ] **Step 6: Run the complete build locally or in CI**

Run:

```bash
gradle testDebugUnitTest assembleDebug
```

Expected: tests PASS and `app/build/outputs/apk/debug/app-debug.apk` exists.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "build: scaffold Android DoIP app"
```

---

### Task 2: DoIP generic frame codec

**Files:**
- Create: `app/src/main/java/com/qujindai/doipmobile/doip/DoipConstants.kt`
- Create: `app/src/main/java/com/qujindai/doipmobile/doip/DoipFrame.kt`
- Create: `app/src/main/java/com/qujindai/doipmobile/doip/DoipCodec.kt`
- Test: `app/src/test/java/com/qujindai/doipmobile/doip/DoipCodecTest.kt`

**Interfaces:**
- Produces `data class DoipFrame(val protocolVersion: Int, val payloadType: Int, val payload: ByteArray)`.
- Produces `object DoipCodec { fun encode(payloadType: Int, payload: ByteArray, protocolVersion: Int = 0x02): ByteArray; fun decode(bytes: ByteArray): DoipFrame }`.
- `DoipConstants.PORT = 13400` and payload type constants `0x0001`, `0x0004`, `0x0005`, `0x0006`, `0x0007`, `0x0008`, `0x8001`, `0x8002`, `0x8003`.

- [ ] **Step 1: Write failing encode/decode tests**

```kotlin
@Test fun encodesVehicleIdentificationRequestHeader() {
    val bytes = DoipCodec.encode(0x0001, byteArrayOf())
    assertArrayEquals(byteArrayOf(0x02, 0xFD.toByte(), 0x00, 0x01, 0, 0, 0, 0), bytes)
}

@Test fun roundTripsDiagnosticPayload() {
    val raw = DoipCodec.encode(0x8001, byteArrayOf(0x0E, 0x80.toByte(), 0x10, 0x00, 0x22, 0xF1.toByte(), 0x90.toByte()))
    val frame = DoipCodec.decode(raw)
    assertEquals(0x8001, frame.payloadType)
    assertArrayEquals(byteArrayOf(0x0E, 0x80.toByte(), 0x10, 0x00, 0x22, 0xF1.toByte(), 0x90.toByte()), frame.payload)
}

@Test(expected = IllegalArgumentException::class)
fun rejectsBadInverseVersion() {
    DoipCodec.decode(byteArrayOf(0x02, 0x02, 0, 1, 0, 0, 0, 0))
}
```

- [ ] **Step 2: Run tests to prove failure**

Run:

```bash
gradle testDebugUnitTest --tests '*DoipCodecTest'
```

Expected: FAIL because `DoipCodec` does not exist.

- [ ] **Step 3: Implement the minimal generic codec**

Implementation requirements:
- 8-byte header.
- big-endian payload type and 32-bit payload length.
- inverse version must equal `protocolVersion xor 0xFF`.
- decoded byte-array length must equal `8 + payloadLength` exactly.
- reject payload lengths above `16 * 1024 * 1024` to avoid accidental allocation abuse.

- [ ] **Step 4: Run tests**

```bash
gradle testDebugUnitTest --tests '*DoipCodecTest'
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/com/qujindai/doipmobile/doip app/src/test/java/com/qujindai/doipmobile/doip/DoipCodecTest.kt
git commit -m "feat: add DoIP frame codec"
```

---

### Task 3: Vehicle identification response parser and UDP discovery

**Files:**
- Create: `app/src/main/java/com/qujindai/doipmobile/doip/VehicleIdentification.kt`
- Create: `app/src/main/java/com/qujindai/doipmobile/network/SocketBinder.kt`
- Create: `app/src/main/java/com/qujindai/doipmobile/network/DoipDiscoveryClient.kt`
- Test: `app/src/test/java/com/qujindai/doipmobile/doip/VehicleIdentificationTest.kt`
- Test: `app/src/test/java/com/qujindai/doipmobile/network/DoipDiscoveryClientTest.kt`

**Interfaces:**
- Produces `data class DoipEntity(val ip: InetAddress, val vin: String, val logicalAddress: Int, val eidHex: String, val gidHex: String, val furtherActionRequired: Int, val vinGidSyncStatus: Int?)`.
- Produces `fun parseVehicleIdentification(payload: ByteArray, ip: InetAddress): DoipEntity`.
- Produces `fun interface SocketBinder { fun bind(socket: DatagramSocket) }` and a parallel `fun bind(socket: Socket)` method through an interface with two overloads.
- Produces `class DoipDiscoveryClient(private val binder: SocketBinder) { suspend fun discover(timeoutMs: Int = 1200): List<DoipEntity> }`.

- [ ] **Step 1: Write parser tests first**

Use a 32-byte response without sync-status and a 33-byte response with it. Example VIN `LTEST123456789012` is exactly 17 ASCII characters.

```kotlin
@Test fun parsesVehicleAnnouncement() {
    val payload = buildList<Byte> {
        addAll("LTEST123456789012".encodeToByteArray().toList())
        add(0x10); add(0x00)
        addAll(byteArrayOf(1,2,3,4,5,6).toList())
        addAll(byteArrayOf(7,8,9,10,11,12).toList())
        add(0x00)
        add(0x00)
    }.toByteArray()
    val e = parseVehicleIdentification(payload, InetAddress.getByName("192.168.0.10"))
    assertEquals("LTEST123456789012", e.vin)
    assertEquals(0x1000, e.logicalAddress)
    assertEquals("010203040506", e.eidHex)
}
```

Also test rejection of payloads shorter than 32 bytes and VINs containing non-printable ASCII.

- [ ] **Step 2: Run parser tests and verify they fail**

```bash
gradle testDebugUnitTest --tests '*VehicleIdentificationTest'
```

Expected: FAIL.

- [ ] **Step 3: Implement parser**

Field offsets are fixed for V1:
- bytes `0..16`: VIN (17 bytes ASCII)
- bytes `17..18`: logical address
- bytes `19..24`: EID
- bytes `25..30`: GID
- byte `31`: further action required
- byte `32` when present: VIN/GID sync status

- [ ] **Step 4: Write a loopback UDP discovery test**

The test starts a local `DatagramSocket(0)` server that receives a DoIP frame and asserts payload type `0x0001`, then replies to the client source port with a `0x0004` frame containing the payload above. Add constructor parameters for destination/port in tests while production defaults remain broadcast/13400.

```kotlin
val client = DoipDiscoveryClient(
    binder = NoOpSocketBinder,
    destination = InetAddress.getLoopbackAddress(),
    port = server.localPort,
)
val entities = runBlocking { client.discover(timeoutMs = 500) }
assertEquals(1, entities.size)
assertEquals("LTEST123456789012", entities.single().vin)
```

- [ ] **Step 5: Implement UDP discovery minimally**

Requirements:
- create `DatagramSocket(null)`, set `reuseAddress = true`, bind to an ephemeral local port, call binder, set `broadcast = true`.
- send `DoipCodec.encode(0x0001, byteArrayOf())`.
- collect valid `0x0004` responses until timeout.
- ignore malformed/non-`0x0004` datagrams instead of crashing the entire discovery operation.
- de-duplicate by `(ip, logicalAddress, eidHex)`.

- [ ] **Step 6: Run tests**

```bash
gradle testDebugUnitTest --tests '*VehicleIdentificationTest' --tests '*DoipDiscoveryClientTest'
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/com/qujindai/doipmobile/doip/VehicleIdentification.kt app/src/main/java/com/qujindai/doipmobile/network app/src/test/java/com/qujindai/doipmobile
git commit -m "feat: discover DoIP vehicles over UDP"
```

---

### Task 4: TCP routing activation and DoIP diagnostic messages

**Files:**
- Create: `app/src/main/java/com/qujindai/doipmobile/doip/RoutingActivation.kt`
- Create: `app/src/main/java/com/qujindai/doipmobile/doip/DiagnosticMessage.kt`
- Create: `app/src/main/java/com/qujindai/doipmobile/network/DoipTcpSession.kt`
- Test: `app/src/test/java/com/qujindai/doipmobile/doip/RoutingActivationTest.kt`
- Test: `app/src/test/java/com/qujindai/doipmobile/doip/DiagnosticMessageTest.kt`
- Test: `app/src/test/java/com/qujindai/doipmobile/network/DoipTcpSessionTest.kt`

**Interfaces:**
- Produces `fun routingActivationRequest(testerAddress: Int = 0x0E80, activationType: Int = 0x00): ByteArray`.
- Produces `data class RoutingActivationResponse(val testerAddress: Int, val entityAddress: Int, val responseCode: Int)` and parser.
- Produces `fun diagnosticRequest(source: Int, target: Int, uds: ByteArray): ByteArray` and `data class DiagnosticPayload(val source: Int, val target: Int, val uds: ByteArray)`.
- Produces `class DoipTcpSession(...): Closeable` with `suspend fun activate(): RoutingActivationResponse` and `suspend fun transact(targetAddress: Int, uds: ByteArray, timeoutMs: Int = 2000): DiagnosticPayload`.

- [ ] **Step 1: Test routing activation bytes**

```kotlin
@Test fun buildsDefaultRoutingActivationRequest() {
    assertArrayEquals(
        byteArrayOf(0x0E, 0x80.toByte(), 0x00, 0, 0, 0, 0),
        routingActivationRequest(0x0E80, 0x00)
    )
}

@Test fun acceptsSuccessCode() {
    val p = byteArrayOf(0x0E,0x80.toByte(),0x10,0x00,0x10,0,0,0,0)
    val r = parseRoutingActivationResponse(p)
    assertEquals(0x10, r.responseCode)
    assertEquals(0x1000, r.entityAddress)
}
```

- [ ] **Step 2: Test diagnostic payload wrapping**

```kotlin
@Test fun wrapsUdsInDiagnosticMessage() {
    val p = diagnosticRequest(0x0E80, 0x1000, byteArrayOf(0x22,0xF1.toByte(),0x90.toByte()))
    assertArrayEquals(byteArrayOf(0x0E,0x80.toByte(),0x10,0x00,0x22,0xF1.toByte(),0x90.toByte()), p)
}
```

- [ ] **Step 3: Run protocol tests and verify failure**

```bash
gradle testDebugUnitTest --tests '*RoutingActivationTest' --tests '*DiagnosticMessageTest'
```

Expected: FAIL.

- [ ] **Step 4: Implement request/response models**

Success is strictly routing activation response code `0x10`. Preserve other response codes in an exception `RoutingActivationDenied(responseCode)` rather than mapping them to a generic network error.

- [ ] **Step 5: Write loopback TCP session test**

Fake server sequence:
1. accept connection;
2. read exactly one generic DoIP frame and assert payload type `0x0005`;
3. reply with `0x0006` success payload;
4. read `0x8001` request and assert embedded UDS `22 F1 90`;
5. reply with `0x8002` acknowledgement;
6. reply with `0x8001` carrying source `0x1000`, target `0x0E80`, UDS `62 F1 90 ...VIN...`.

Client assertion: `activate().responseCode == 0x10`, then `transact(...).uds` starts with `62 F1 90`.

- [ ] **Step 6: Implement stream-safe frame reads**

`DoipTcpSession` must not assume one TCP read equals one DoIP frame. Implement `readExactly(8)`, decode the header, then `readExactly(payloadLength)`.

When waiting for a diagnostic response:
- accept/ignore `0x8002` ACK;
- throw a typed exception for `0x8003` NACK;
- return the matching `0x8001` response addressed to the tester.

- [ ] **Step 7: Run tests**

```bash
gradle testDebugUnitTest --tests '*RoutingActivationTest' --tests '*DiagnosticMessageTest' --tests '*DoipTcpSessionTest'
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/com/qujindai/doipmobile/doip app/src/main/java/com/qujindai/doipmobile/network/DoipTcpSession.kt app/src/test/java/com/qujindai/doipmobile
git commit -m "feat: add DoIP TCP routing session"
```

---

### Task 5: Read-only UDS VIN codec

**Files:**
- Create: `app/src/main/java/com/qujindai/doipmobile/uds/Uds.kt`
- Test: `app/src/test/java/com/qujindai/doipmobile/uds/UdsTest.kt`

**Interfaces:**
- Produces `object Uds { val readVinRequest: ByteArray; fun parseVinResponse(bytes: ByteArray): String }`.
- Produces `class UdsNegativeResponse(val requestService: Int, val nrc: Int) : Exception(...)`.

- [ ] **Step 1: Write tests first**

```kotlin
@Test fun readVinRequestIs22F190() {
    assertArrayEquals(byteArrayOf(0x22, 0xF1.toByte(), 0x90.toByte()), Uds.readVinRequest)
}

@Test fun parsesPositiveVinResponse() {
    val bytes = byteArrayOf(0x62,0xF1.toByte(),0x90.toByte()) + "LTEST123456789012".encodeToByteArray()
    assertEquals("LTEST123456789012", Uds.parseVinResponse(bytes))
}

@Test(expected = UdsNegativeResponse::class)
fun surfacesNegativeResponse() {
    Uds.parseVinResponse(byteArrayOf(0x7F,0x22,0x31))
}
```

Also test positive response with the wrong DID and VIN not exactly 17 bytes.

- [ ] **Step 2: Run and verify failure**

```bash
gradle testDebugUnitTest --tests '*UdsTest'
```

Expected: FAIL.

- [ ] **Step 3: Implement only the required UDS functionality**

No generic UDS service framework in V1. Implement only:
- request `22 F1 90`;
- positive response `62 F1 90 + 17 ASCII VIN`;
- negative response `7F 22 NRC`.

- [ ] **Step 4: Run tests**

```bash
gradle testDebugUnitTest --tests '*UdsTest'
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/com/qujindai/doipmobile/uds app/src/test/java/com/qujindai/doipmobile/uds
git commit -m "feat: add read-only VIN UDS codec"
```

---

### Task 6: Android Ethernet selection and per-socket network binding

**Files:**
- Create: `app/src/main/java/com/qujindai/doipmobile/network/AndroidEthernetSelector.kt`
- Modify: `app/src/main/java/com/qujindai/doipmobile/network/SocketBinder.kt`
- Create: `app/src/test/java/com/qujindai/doipmobile/network/Ipv4BroadcastTest.kt`

**Interfaces:**
- Produces `data class EthernetEndpoint(val network: Network, val ipv4: Inet4Address, val prefixLength: Int, val broadcast: Inet4Address)`.
- Produces `class AndroidEthernetSelector(private val connectivityManager: ConnectivityManager)` with `fun current(): EthernetEndpoint?`.
- `EthernetEndpoint` implements/provides `SocketBinder` behavior by calling `network.bindSocket(datagramSocket)` and `network.bindSocket(socket)`.

- [ ] **Step 1: Write pure IPv4 broadcast calculation tests**

Extract `fun ipv4Broadcast(address: Inet4Address, prefixLength: Int): Inet4Address` so it is JVM-testable.

```kotlin
@Test fun calculates24Broadcast() {
    assertEquals(
        "192.168.42.255",
        ipv4Broadcast(InetAddress.getByName("192.168.42.21") as Inet4Address, 24).hostAddress
    )
}
```

Also test `/16` and `/32`.

- [ ] **Step 2: Implement network selection**

Algorithm:
1. iterate `connectivityManager.allNetworks`;
2. require `NetworkCapabilities.TRANSPORT_ETHERNET` only; do not require validated internet;
3. read `LinkProperties.linkAddresses` and select the first IPv4 address;
4. compute subnet broadcast from prefix length;
5. return null if no Ethernet IPv4 exists.

Do not use `bindProcessToNetwork`.

- [ ] **Step 3: Connect production discovery/session construction to the endpoint binder and broadcast address**

Production discovery destination becomes `endpoint.broadcast`; TCP destination remains the discovered entity IP. Both sockets are individually bound to `endpoint.network` before connect/send.

- [ ] **Step 4: Run JVM tests and Android compile**

```bash
gradle testDebugUnitTest assembleDebug
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/com/qujindai/doipmobile/network app/src/test/java/com/qujindai/doipmobile/network
git commit -m "feat: bind DoIP sockets to Android Ethernet"
```

---

### Task 7: Diagnostic repository and end-to-end fake-server test

**Files:**
- Create: `app/src/main/java/com/qujindai/doipmobile/domain/DiagnosticRepository.kt`
- Test: `app/src/test/java/com/qujindai/doipmobile/domain/DiagnosticRepositoryTest.kt`

**Interfaces:**
- Produces `data class VehicleSessionInfo(val entity: DoipEntity, val vin: String)`.
- Produces `sealed interface DiagnosticEvent` for human-readable raw-console events.
- Produces `suspend fun discover(): List<DoipEntity>` and `suspend fun readVin(entity: DoipEntity): VehicleSessionInfo`.

- [ ] **Step 1: Write orchestration test with fakes**

Use fake discovery/session factories instead of Android types. Verify call order:
1. discovery returns entity logical address `0x1000`;
2. session activation occurs;
3. session `transact(0x1000, Uds.readVinRequest)` occurs;
4. repository returns 17-character VIN.

Also verify a routing activation denial does not attempt a UDS request.

- [ ] **Step 2: Run and verify failure**

```bash
gradle testDebugUnitTest --tests '*DiagnosticRepositoryTest'
```

Expected: FAIL.

- [ ] **Step 3: Implement repository**

Repository owns no Android lifecycle. It receives factories/interfaces in its constructor and emits events through a callback `(DiagnosticEvent) -> Unit` so the ViewModel can append console rows.

Events must include at minimum:
- Ethernet selected;
- discovery TX/RX;
- routing activation TX/RX;
- UDS TX `22 F1 90`;
- DoIP diagnostic RX;
- parsed VIN;
- typed error.

Raw bytes should be formatted uppercase hex with spaces.

- [ ] **Step 4: Run all protocol/domain tests**

```bash
gradle testDebugUnitTest
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/com/qujindai/doipmobile/domain app/src/test/java/com/qujindai/doipmobile/domain
git commit -m "feat: orchestrate DoIP VIN diagnosis"
```

---

### Task 8: Compose UI and state machine

**Files:**
- Create: `app/src/main/java/com/qujindai/doipmobile/ui/DoipUiState.kt`
- Create: `app/src/main/java/com/qujindai/doipmobile/ui/DoipViewModel.kt`
- Create: `app/src/main/java/com/qujindai/doipmobile/ui/DoipScreen.kt`
- Modify: `app/src/main/java/com/qujindai/doipmobile/MainActivity.kt`
- Test: `app/src/test/java/com/qujindai/doipmobile/ui/DoipViewModelTest.kt`

**Interfaces:**
- `DoipUiState` fields: `ethernetConnected`, `localIp`, `isBusy`, `entities`, `selectedEntity`, `vin`, `statusText`, `logs`.
- `DoipViewModel` commands: `refreshEthernet()`, `discover()`, `selectEntity(DoipEntity)`, `readVin()`, `clearLog()`.

- [ ] **Step 1: Write ViewModel state tests**

At minimum:
- initial state is disconnected and idle;
- discover toggles busy and publishes returned entities;
- reading VIN publishes VIN and returns to idle;
- exceptions publish a user-visible status and append an error log without crashing.

- [ ] **Step 2: Implement ViewModel using `viewModelScope`**

Never do network I/O on the main thread. All discovery/TCP calls run on `Dispatchers.IO` or are already suspend functions that switch internally.

- [ ] **Step 3: Implement a single-screen Compose UI**

Screen sections:
1. **Ethernet** — connected/disconnected, local IPv4, Refresh button.
2. **DoIP Discovery** — Discover button and selectable entity cards showing IP, VIN from announcement if present, logical address, EID.
3. **Diagnosis** — `Read VIN (22 F190)` button; disabled unless an entity is selected and no request is running.
4. **Result** — large VIN text.
5. **Raw Console** — timestamp/direction/type/hex rows and Clear Log.

There must be no buttons for Clear DTC, Reset ECU, Security Access, Routine Control, WriteDataByIdentifier, RequestDownload, TransferData, or flashing.

- [ ] **Step 4: Wire MainActivity**

Construct `ConnectivityManager`, `AndroidEthernetSelector`, repository dependencies, then create the ViewModel through a factory. Keep this manual in V1; do not add Hilt/Koin.

- [ ] **Step 5: Run tests and build**

```bash
gradle testDebugUnitTest assembleDebug
```

Expected: PASS and debug APK generated.

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/com/qujindai/doipmobile/ui app/src/main/java/com/qujindai/doipmobile/MainActivity.kt app/src/test/java/com/qujindai/doipmobile/ui
git commit -m "feat: add DoIP diagnostic UI"
```

---

### Task 9: Final integration gate, APK artifact, and operator documentation

**Files:**
- Modify: `README.md`
- Modify: `.github/workflows/android.yml`
- Create: `docs/TESTING.md`
- Create: `docs/VEHICLE_TEST_CHECKLIST.md`

**Interfaces:**
- CI output: `app-debug.apk` uploaded as artifact `doip-mobile-debug-apk`.
- Human vehicle-test contract documents exact preconditions and expected packet sequence.

- [ ] **Step 1: Add README usage path**

Document:

```text
Vehicle DoIP OBD/ENET -> USB-C Ethernet -> Android
Open DoIP Mobile -> Refresh Ethernet -> Discover -> select entity -> Read VIN
```

State explicitly that V1 is read-only and that OBD DoIP pinout/activation compatibility must be verified for the target vehicle/cable before connection.

- [ ] **Step 2: Add local fake-server test instructions**

`docs/TESTING.md` must include:

```bash
gradle testDebugUnitTest
gradle assembleDebug
```

and describe the fake UDP/TCP server sequence used by tests.

- [ ] **Step 3: Add controlled vehicle test checklist**

Checklist order:
1. verify the cable is intended for the exact vehicle DoIP pinout;
2. connect vehicle side;
3. connect USB-C to phone;
4. confirm Android reports Ethernet;
5. app Refresh shows IPv4;
6. Discover receives at least one `0x0004` response;
7. activate selected DoIP entity and record `0x0006` result;
8. send only `22 F1 90`;
9. verify positive `62 F1 90 + VIN` or capture NRC/error;
10. disconnect and export/capture console evidence manually if needed.

- [ ] **Step 4: Run the full quality gate**

```bash
gradle clean testDebugUnitTest assembleDebug
```

Expected: all tests PASS and APK exists.

- [ ] **Step 5: Verify CI on `main`**

Expected GitHub Actions sequence: checkout -> JDK -> Gradle -> unit tests PASS -> assemble PASS -> artifact upload PASS.

- [ ] **Step 6: Commit**

```bash
git add README.md docs .github/workflows/android.yml
git commit -m "docs: add DoIP V1 validation procedure"
```

---

## Acceptance Criteria

V1 is complete only when all of the following are true:

1. `gradle clean testDebugUnitTest assembleDebug` passes on JDK 17 / Gradle 9.5.0.
2. GitHub Actions passes on `main` and publishes `doip-mobile-debug-apk`.
3. Generic DoIP frame tests validate header inversion, payload length, and big-endian encoding.
4. UDP loopback test proves `0x0001 -> 0x0004` discovery parsing.
5. TCP loopback test proves `0x0005 -> 0x0006 (0x10)` routing activation.
6. TCP loopback test proves `0x8001` UDS transport while tolerating `0x8002` ACK.
7. UDS test proves `22 F1 90 -> 62 F1 90 + 17-char VIN` parsing and typed `7F 22 NRC` failure.
8. Android implementation selects `TRANSPORT_ETHERNET` without requiring internet validation and binds only the individual DoIP sockets.
9. APK exposes no write/programming diagnostic functions.
10. On a compatible real DoIP vehicle/cable, the operator flow can progress from Ethernet detection through VIN read without CAN, root, or a proprietary VCI.

## Self-Review

- Spec coverage: physical connection, Android Ethernet binding, DoIP discovery, routing activation, UDS VIN, raw console, read-only safety, tests, and APK artifact are each mapped to a task.
- Placeholder scan: no `TBD`, `TODO`, unspecified error-handling step, or undefined follow-on task remains.
- Type consistency: `DoipEntity`, `SocketBinder`, `DoipTcpSession`, `Uds`, repository and UI state names are consistent across tasks.
- Scope: V1 deliberately excludes ECU enumeration beyond discovered DoIP entities, DTC services, flashing, security access, and OEM-specific routing/security behavior; those require separate post-V1 specs.
