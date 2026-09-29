# SK6 SDK Guide

The SK6 main board exposes hardware control to Android applications through `SdkApiJar.aar`:
the app binds the pre-installed `com.sdkapi.sdkapiservice` service and calls device information,
display, system control, power-on/off schedule, watchdog, GPIO / UART / CAN / Flash, scale and
common-event APIs over AIDL.

This document targets application integration and complete-machine / board verification.
It covers basic SDK usage and **how to control the bottom LED strip (WS2812B board) over the serial port**.

## 1. Repository contents

| File | Description |
|---|---|
| `SdkApiJar_V1.250915.0914.aar` | SDK interface library, package `com.sdkapi`, versionName `4.1.0.250915.0914` |
| `SdkApi Guidance document-CN.pdf` | Chinese API manual (185 pages), per-interface prototype / parameter table / return value / code sample |
| `SdkApi Guidance document-EN.pdf` | English API manual |

`apiVersion = 250915.0914` (`com.sdkapi.common.Const.apiVersion`).

## 2. Requirements

- Android SDK: `minSdkVersion 19`, `targetSdkVersion 30`.
- The service package `com.sdkapi.sdkapiservice` must be pre-installed on the device and bindable
  (the aar `AndroidManifest.xml` already declares `<queries>` for it).
- Available functions depend on the board firmware and service version. Unsupported calls return `API_RES_UNSUPPORT`.

## 3. Integration

1. Copy the aar into `app/libs/`.
2. Add the reference in `app/build.gradle`:

   ```gradle
   android {
       sourceSets { main { jniLibs.srcDirs = ['libs'] } }
   }
   dependencies {
       implementation files('libs/SdkApiJar_V1.250915.0914.aar')
   }
   ```

3. Register and bind `SdkApi` in the Activity, then obtain each sub-module from the `SdkApi` instance.

## 4. Initialization and release

```java
// Bind the service; call once, e.g. in onCreate
SdkApi.newInstance(this);

// After newInstance() the service needs roughly 2 s to connect, so check the status before use
boolean connected = SdkApi.getInstance().getServiceConnectionStatus();

// Unbind on exit
SdkApi.getInstance().unBindApiSdkService();
```

## 5. API modules

All sub-modules are obtained from `SdkApi.getInstance()`:

| Entry point | Class | Scope |
|---|---|---|
| `DeviceInfo()` | `ApiDeviceInfo` | API / service / firmware / system version, chip name, serial number, RAM and storage size, MCU version, screen density, system properties, Ethernet MAC, LN |
| `Display()` | `ApiDisplay` | Screen rotation, screen on/off, brightness, backlight inverse and PWM, resolution, LVDS mode / channel / Even-ODD, custom timing and MIPI commands, graphic display region |
| `SystemCtrl()` | `ApiSystemCtrl` | Reboot, screenshot, status bar / navigation bar visibility and gestures, normal and low-power standby, fan, silent APK install/uninstall, storage format, factory reset, network priority |
| `TimerSwitch()` | `ApiTimerOnoff` | Timer on/off switch, on/off times (daily / by date / weekly), standby mode on timer off |
| `Watchdog()` | `ApiWatchdog` | Watchdog owner and on/off, timeout, feed |
| `SystemTime()` | `ApiSystemTime` | Set system time, read RTC, auto time-sync switch |
| `Hardware()` | `ApiHardware` | GPIO read/write, Flash read/write, board running LED, keypad red/green LEDs, USB port power, speaker output, **serial port**, CAN, MCU communication |
| `Other()` | `ApiOther` | SIM ICCID, IMEI/CTEI, shell command, eMMC lifetime, overlay DRM |
| `Scale()` | `ApiScale` | Scale listener register/unregister, scale events |
| `CommonEvent()` | `ApiCommonEvent` | Common event listener and action dispatch (actionId command set: manual chapter 11) |

Calling conventions:

- Synchronous APIs return `boolean` or `int`; for `int`, use the `ApiTool` constants:
  `API_RES_OK = 0`, `API_RES_FAILED = -1`, `API_RES_UNSUPPORT = -2`.
- Asynchronous APIs take a `ResultCallback<ResultInfo<T>>` and deliver results through `onMsg(ResultInfo)`.
  `ResultInfo` exposes `getStatus()`, `getVal()` and `getArrayByte()`; multi-field results are read from
  `getArrayByte()` by index as defined in the manual.
- Constants (GPIO pins, screen IDs, Flash types, and so on) live in the inner classes of `com.sdkapi.common.ApiTool`.

## 6. Serial port APIs (`Hardware()`)

| API | Description |
|---|---|
| `int openSerial(String serialName, int baud)` | Opens the port in 8N1 only; returns the port handle |
| `int openSerial(String serialName, int baud, int dataBits, int parity, int stopBits)` | General form. `dataBits` 5/6/7/8; `parity` 0=NONE, 1=ODD, 2=EVEN; `stopBits` 1/2 |
| `int closeSerial(int serial)` | 0 = closed successfully, other values = failure |
| `int writeSerial(int serial, byte[] buf, int len)` | Returns the number of bytes actually written |
| `int readSerial(int serial, byte[] buf, int len)` | Returns the number of bytes actually read |

The port is opened by the SDK service, so the app works with the returned handle and never touches the device node directly.

## 7. Bottom LED control

The bottom LED strip is driven by an STC8G1K08A LED board: the main board sends binary frames over UART,
and the LED board outputs WS2812B data on pin 8 (P3.3).
The current protocol version is `VER = 1`. The firmware ships with a test default of 11 LEDs and accepts
1–1024 at runtime, so the app must not assume a fixed count — query the status after connecting, or send the
count actually installed on the product.

### 7.1 Serial settings

| Item | Value |
|---|---|
| Device node | `/dev/ttyS1` |
| Baud rate | `115200` |
| Data bits | `8` |
| Parity | None |
| Stop bits | `1` |
| Flow control | None |
| Summary | `8N1`, binary bytes — never ASCII text |

Wiring: board TX → MCU pin 5 (P3.0/RXD), board RX → MCU pin 6 (P3.1/TXD), GND must be common,
3.3V powers the MCU. The WS2812B strip has its own 5V supply; 3.3V, 5V and GND must share a common ground.

### 7.2 Frame layout

```text
AA 55 | VER | CMD | LEN | PAYLOAD | CRC16_L | CRC16_H
 header  ver   cmd    len    data       checksum (low byte first)
```

The header is always `AA 55` and `VER` is always `01`. CRC16 covers `VER, CMD, LEN, PAYLOAD` only —
not the header and not the checksum itself:

```text
crc = 0xFFFF
for byte in VER + CMD + LEN + PAYLOAD:
    crc ^= byte
    repeat 8:
        crc = (crc & 1) ? ((crc >> 1) ^ 0xA001) : (crc >> 1)
send the low byte of crc first, then the high byte
```

The response command code is the request command code plus `0x80`; a normal success reply carries the single
payload byte `00`. Example — successful `SET_STATIC` reply:

```text
AA 55 01 81 01 00 50 60
```

The app must validate the complete reply frame, response command code, length, status code and CRC.
Checking that the port merely received something is not enough.

### 7.3 Command set

| CMD | SDK method | PAYLOAD | Purpose |
|---:|---|---|---|
| `01` | `setStatic` | `R G B BRIGHTNESS` | All LEDs a fixed colour (also switches to static mode and turns the display on) |
| `02` | `setEffect` | `MODE R G B BRIGHTNESS SPEED DIRECTION` | Dynamic effect |
| `03` | `setBrightness` | `BRIGHTNESS` | Change brightness |
| `04` | `setLedCount` | `COUNT_L COUNT_H` (little endian) | Set LED count |
| `05` | `setSpeed` | `SPEED` | Change effect speed |
| `06` | `setDirection` | `DIRECTION` | Change direction |
| `07` | `setPower` | `00` / `01` | Display off / restore |
| `08` | `getStatus` | none | Read current running state |
| `09` | `getInfo` | none | Read protocol version and capabilities |
| `0A` | `saveDefault` | none | Save current state as power-on default (writes EEPROM) |
| `0B` | `getDefault` | none | Read the saved power-on default |

Direction: `00` forward, `01` reverse, `02` bounce — it only affects dynamic modes. Static colour needs no
direction, and MODE 4 accepts only `02`. Speed ranges from `01` to `FF`; higher is faster
(suggested steps: slow `20`, medium `80`, fast `C0`, fastest `FF`).

### 7.4 Colour byte order (important)

The protocol always carries the colour as three bytes in **`R G B`** order, each `00`–`FF`.

For example, if the Android colour picker yields `#FF8000`, send `R=FF, G=80, B=00`.
The firmware converts the protocol `R G B` into the **G R B** order the WS2812B board requires,
so **the Android side must not swap the colour channels again**.

| Colour | R | G | B | Value |
|---|---:|---:|---:|---|
| Off | `00` | `00` | `00` | `000000` |
| Red | `FF` | `00` | `00` | `FF0000` |
| Green | `00` | `FF` | `00` | `00FF00` |
| Blue | `00` | `00` | `FF` | `0000FF` |
| Yellow | `FF` | `FF` | `00` | `FFFF00` |
| Purple | `FF` | `00` | `FF` | `FF00FF` |
| Cyan | `00` | `FF` | `FF` | `00FFFF` |
| White | `FF` | `FF` | `FF` | `FFFFFF` |
| Orange | `FF` | `80` | `00` | `FF8000` |

### 7.5 Brightness (1 byte)

Brightness is a separate byte sent together with the colour bytes. The firmware scales R, G and B by it,
so the colour ratio stays the same:

| Value | Approx. | Use |
|---:|---:|---|
| `00` | 0% | Output off |
| `40` | 25% | Low brightness |
| `80` | 50% | Medium brightness |
| `C0` | 75% | High brightness |
| `FF` | 100% | Maximum brightness |

The current firmware applies no gamma correction, so an app slider value is not perfectly linear to perceived
brightness. A linear percentage is fine for the first app version; add a UI-side gamma curve later if needed.

### 7.6 Common commands

```text
Set brightness 50%:   AA5501030180F1E8
Set speed 128:        AA550105018011E9
Set forward:          AA5501060100E049
Set reverse:          AA55010601012189
Set bounce:           AA55010601026188
Display off:          AA5501070100B189
Display restore:      AA55010701017049
Query status:         AA5501080027C0
Query info (fw):      AA550109002650
Save power-on default:AA55010A0026A0
Query default:        AA55010B002730
Set 11 LEDs:          AA550104020B00BE00
Set 60 LEDs:          AA550104023C00A830

Static red:           AA55010104FF0000FF8B85
Static green:         AA5501010400FF00FF8BA1
Static blue:          AA550101040000FFFFFA61
Static white 50%:     AA55010104FFFFFF80BBA5
Static white 100%:    AA55010104FFFFFFFFFA45
Static orange 78%:    AA55010104FF8000C8CBBB

MODE 2 rainbow flow:  AA5501020702000000FF80000B74
MODE 3 cyan trail:    AA550102070300FFFFC84000DE7A
MODE 4 rainbow bounce:AA5501020704000000FF8002ECB5
```

Mode notes:

- MODE 2, one-way rainbow flow: all LEDs stay lit, the firmware colour wheel is used, `R/G/B` are kept but do not change the rainbow.
- MODE 3, single-colour fading trail: all LEDs visible, `R/G/B` is the trail base colour, the head is full bright and the tail half/low bright; supports forward, reverse and bounce.
- MODE 4, rainbow bounce: all LEDs stay lit on the built-in colour wheel, `DIRECTION` must be `02`, the effect reverses automatically at each end.

### 7.7 Status and capability replies

The success payload of `getStatus` (`08`) is 13 bytes:

```text
STATUS MODE POWER BRIGHTNESS SPEED DIRECTION COUNT_L COUNT_H R G B PHASE_L PHASE_H
Example: AA5501880D000101FF80000B00FFFFFF0000F831
  → STATUS=00 MODE=01 POWER=01 BRIGHTNESS=FF SPEED=80 DIRECTION=00 COUNT=11 RGB=FFFFFF PHASE=0000
```

The payload of `getInfo` (`09`) is `STATUS VERSION MAX_COUNT(2B) CAPABILITY`:

```text
Request: AA550109002650
Reply:   AA550189050001000403DA67
  → STATUS=00 VERSION=01 MAX_COUNT=0x0400(1024) CAPABILITY=03
Capability bits: bit0=1 streaming light effects, bit1=1 save power-on default
```

`getDefault` (`0B`) feeds the "power-on default" screen and is not a substitute for `getStatus`; the app should
keep both `currentState` (actual running state) and `defaultState` (saved default state).

### 7.8 Code sample

Frame builder, matching the CRC definition in 7.2:

```java
private static byte[] crcAppend(byte[] body) {
    int crc = 0xFFFF;
    for (byte b : body) {
        crc ^= (b & 0xFF);
        for (int i = 0; i < 8; i++) {
            crc = ((crc & 1) != 0) ? ((crc >> 1) ^ 0xA001) : (crc >> 1);
        }
    }
    byte[] frame = new byte[body.length + 4];
    frame[0] = (byte) 0xAA;
    frame[1] = (byte) 0x55;
    System.arraycopy(body, 0, frame, 2, body.length);
    frame[frame.length - 2] = (byte) (crc & 0xFF);   // CRC_L
    frame[frame.length - 1] = (byte) (crc >> 8);      // CRC_H
    return frame;
}

// body = VER + CMD + LEN + PAYLOAD
static byte[] buildFrame(int cmd, byte[] payload) {
    byte[] body = new byte[3 + payload.length];
    body[0] = 0x01;                 // VER
    body[1] = (byte) cmd;
    body[2] = (byte) payload.length;
    System.arraycopy(payload, 0, body, 3, payload.length);
    return crcAppend(body);
}

/** Pass the colour as a standard 0xRRGGBB int; it is split into R G B without channel swapping */
static byte[] staticColorFrame(int rgb, int brightness) {
    return buildFrame(0x01, new byte[]{
        (byte) (rgb >> 16), (byte) (rgb >> 8), (byte) rgb, (byte) brightness});
}
```

Controlling the bottom LED strip through `/dev/ttyS1`:

```java
SdkApi.newInstance(this);                       // then wait ~2 s for the service to connect

ApiHardware hw = SdkApi.getInstance().Hardware();
int fd = hw.openSerial("/dev/ttyS1", 115200);   // equivalent: openSerial("/dev/ttyS1", 115200, 8, 0, 1)

// Init sequence: GET_INFO → GET_DEFAULT → GET_STATUS, then refresh the UI from the device state
byte[] req = buildFrame(0x09, new byte[]{});     // AA550109002650
hw.writeSerial(fd, req, req.length);
byte[] rsp = new byte[64];
int len = hw.readSerial(fd, rsp, rsp.length);    // validate CMD = request + 0x80, LEN, STATUS, CRC

byte[] color = staticColorFrame(0xFF8000, 0xC8);   // orange 78% → AA55010104FF8000C8CBBB
hw.writeSerial(fd, color, color.length);

hw.closeSerial(fd);
```

Sends must be **serialised**: one frame out → wait for the complete ACK → then the next frame.
Use a single locked send queue for the port, and throttle colour-slider changes in the SDK layer so only the
last value is sent. `setStatic`, `setEffect` and `setLedCount` refresh the whole strip before ACKing, so do not
set a short timeout with many LEDs — 500 ms is a reasonable base, increase it with the LED count.

### 7.9 Power-on default and error handling

- Normal setting commands change only the running state and **do not write EEPROM**. Send `0A` only when the
  user explicitly taps "set as power-on default". The saved content is LED count, mode, power state, RGB,
  brightness, speed and direction — the animation phase is not saved. After repowering, the firmware starts
  from the saved default and dynamic effects restart at phase 0.
- Do not send `POWER=0` automatically on serial disconnect, page exit or connection switching — that turns the
  strip off. Send the off command only when the user taps "turn off light". To keep the strip off after shutdown,
  turn it off and then send `SAVE_DEFAULT`.
- Status codes:

  | Status | App behaviour |
  |---:|---|
  | `00` | Update local state, continue |
  | `01` | Report firmware protocol version mismatch |
  | `02` | Do not retry; correct the parameters and send again |
  | `03` | Check the firmware version or command code |
  | `04` | Check serial noise, frame framing and the CRC implementation |

- After a timeout, do not fire a concurrent resend: clear the receive buffer and re-send `GET_STATUS`;
  on repeated failures close and reopen the port, then re-init as `GET_INFO → GET_DEFAULT → GET_STATUS`.

### 7.10 Verification notes and acceptance checklist

- If `getServiceConnectionStatus()` returns `false`, wait about 2 s and retry before declaring the serial port broken.
- `openSerial` returns a handle; `writeSerial` / `readSerial` return the actual length. On `<= 0`, check the node, baud rate and service status.
- Channel order is `R G B`. If red and green appear swapped or hues are off, check whether the caller swapped the channels an extra time before blaming the firmware.
- A serial debug tool must use hex send and hex display — no spaces, carriage returns or line feeds.
  The query frame `AA550109002650` only returns information and never lights anything; lighting requires `SET_STATIC` or `SET_EFFECT`.
- The bottom strip (`/dev/ttyS1`, MCU board) is a different object from the board running LED and the keypad
  red/green LEDs (`setRunningLedOnoff`, `setKeyBoardLedROnoff/GOnoff`) — do not mix them up.

After the first integration, verify at least: 115200 8N1 opens, `GET_INFO` returns `VERSION=01 / MAX_COUNT=1024 / CAPABILITY=03`,
setting 11 LEDs works, static red/green/blue/white are correct, brightness 25% / 50% / 100% behave,
MODE 2, 3 and 4 render correctly, speed and direction take effect, `POWER=0` turns off and `POWER=1` restores,
`SAVE_DEFAULT` survives a power cycle, the app distinguishes `currentState` from `defaultState` after reconnect,
and no command is sent with spaces or ASCII encoding.

## 8. Full API reference

Per-interface parameter tables, return values and code samples are in `SdkApi Guidance document-EN.pdf`
(and the CN edition): 1 initialization, 2 device info, 3 display, 4 system control, 5 timer on/off,
6 watchdog, 7 system time, 8 hardware (serial / CAN / GPIO / Flash), 9 other, 10 scale, 11 common event;
API constants and error-code tables are in the appendix.
The LED field definitions in chapter 7 above come from the LED board firmware's Android integration document;
chapter 7 merges that protocol with the SK6 serial port API.
