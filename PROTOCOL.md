# V380 (Macro-Video) camera: native LAN protocol

Clean-room write-up of the local-network protocol spoken by a dual-lens pan/tilt
Wi-Fi camera sold as **S300** and managed by the **V380 Pro** Android app
(`com.macrovideo.v380pro`). It lets an open client authenticate, receive live
video and audio, talk back and pan/tilt the camera directly on the LAN, without
the phone app or the vendor cloud.

Each section says how it was established:

- **Validated** — exercised against a real camera.
- **Static** — read from the shipped app (Java/smali or ARM64 native libraries);
  not yet exercised on a camera.

## Tested device

| Item | Value |
|---|---|
| Commercial name | S300 (dual-lens, pan/tilt, Wi-Fi) |
| App QR prefix on the label | `V380^` |
| Hardware | `HsAkQQVL` |
| Firmware | `Hw_HsAkQQVL_WF_QQ_20240412`, application `AppEV3L_V2_V1.0.9.10_20250731` |
| Reported device version | 32 |
| Reference app | V380 Pro 2.2.70 (version code 211) |

Related V380 products and other firmware versions can differ.

## Conventions

All integers are little endian. Unlisted bytes in a request are zero.
Authentication and streaming use **separate TCP connections to port 8800**.
TCP reads can split or combine protocol messages: keep buffered bytes across
reads. "Device identifier" is the numeric ID printed on the label / encoded in
the QR code.

## Discovery (static, unconfirmed)

The app broadcasts a 128-byte UDP request beginning `NVDEVSEARCH^100` to port
10008 and listens on 10009. In a reply, field 12 carries the device identifier.
This camera did not answer UDP discovery in our tests; a client should instead
remember the camera's MAC address after a successful login and find it again
through the ARP/neighbour table. An open TCP port 8800 alone is not enough to
identify the device.

## Authentication (validated)

Send a 520-byte request:

| Offset | Size | Meaning |
|---|---|---|
| 0 | u32 | Command 1167 |
| 4 | u32 | LAN request kind 120 |
| 8 | u8 | Request protocol 31 |
| 9 | u32 | Application field; observed value 10 |
| 13 | u32 | Device identifier |
| 17 | 32 bytes | Application timestamp, ASCII |
| 49 | 32 bytes | Username, NUL padded |
| 81 | 64 bytes | Password envelope, NUL padded |

**Password envelope.** Pick a random 16-character ASCII key *K*. Encrypt the
UTF-8 password with AES-128-ECB and PKCS#7 padding under the fixed application
key (`APP_KEY`, see [Constants](#constants)), then encrypt that result again with
AES-128-ECB / PKCS#7 under *K*. The envelope is *K* followed by the binary
ciphertext. An empty password gives 32 bytes of ciphertext; a 24-byte password
gives 48.

The 256-byte response starts with command 1168 and result 1001 on success.
Byte 12 is the device version, u32 at 13 the **authentication ticket**, u32 at 17
the session identifier. Close this connection after reading the full response.

## Live stream (validated)

Open a new connection and send a 256-byte request:

| Offset | Size | Meaning |
|---|---|---|
| 0 | u32 | Command 301 |
| 4 | u32 | Device identifier |
| 8 | u32 | Play channel; 0 |
| 12 | u16 | Requested communication version: 21 for device version 32 |
| 14 | u32 | Authentication ticket |
| 22 | u32 | Audio flag: 4096 off, 4097 on |
| 26 | u32 | Stream quality; 0 |
| 30–32 | 3 bytes | Each set to 1 |

The 32-byte response starts with command 401 and result 1001. u16 at 8 is the
negotiated communication version (**20** on this camera, despite requesting
21); width, height and maximum packet size are u32 at 10, 14 and 18. Bytes 22,
23 and 24 give the audio frequency enum (1 = 16 kHz, otherwise 8 kHz), sample
bits and channels; observed 0, 16, 1.

Then send command 303 in a 256-byte buffer with u16 `0x3001` at offset 4. Media
arrives on the same connection.

Observed video: HEVC, 640 × 720, about 16 fps. Both lenses are stacked
vertically in one frame (fixed view on top, movable view below).

### Keepalive and audio toggle (validated)

During streaming, send 16 bytes: u32 command 8449 (`0x2101`) at 0, u32 play
channel at 4, u32 audio flag (4096/4097) at 8, zero at 12. The same packet is the
periodic keepalive: always send the *current* audio flag, or a keepalive will
silently switch audio off.

## Media framing and decryption (validated for version 20)

Fragments have a 12-byte header: magic `0x7f` at 0, type at 1, total fragment
count u16 at 3, fragment index u16 at 5, payload length u16 at 7. Reassemble
consecutive indexes from zero; reassemble audio and video separately because
they interleave.

A completed frame begins with a 16-byte inner header: frame ID u32 at 0, frame
type u16 at 4, frame rate (or audio rate field) u16 at 6, timestamp u64 at 8.
The bytes after it are encrypted. Frame type `0x28` is HEVC video; audio types
are listed below.

**Media key** (16 bytes):
`u32le(ticket) || u64le(MEDIA_KEY_A) || u32le(MEDIA_KEY_B)` (see
[Constants](#constants)).

- Video, negotiated version 20: AES-128-ECB-decrypt 64 bytes, skip 16, repeat
  while at least 64 bytes remain; a final remainder shorter than 64 bytes is
  left as is.
- Video, negotiated version 21 (static only): decrypt the first
  `min(2048, length rounded down to 16)` bytes.
- Audio, both versions: decrypt every complete 16-byte block; leave a partial
  tail intact.

## Pan and tilt (validated)

Send a 16-byte packet on the live stream connection:

| Offset | Size | Meaning |
|---|---|---|
| 0 | u32 | Command 170 |
| 4 | u16 | 1000 |
| 6 | u16 | 1000 |
| 8 | u16 | Pan: 1000 stop, 1001 left, 1002 right |
| 10 | u16 | Tilt: 1000 stop, 1003 up, 1004 down |
| 12 | u16 | PTZ channel; 0 |
| 14 | u16 | Step; 1 |

A stop packet sets both directions to 1000. The camera has no per-movement
timeout: the client must send stop itself and should bound movement duration.
Movement of the lower (movable) view was confirmed by image registration; no
absolute angle calibration is claimed.

## Audio

### Receive (validated)

With the audio flag on, the camera sends frame type 22: 256-byte **IMA ADPCM**
blocks (Microsoft layout: signed 16-bit predictor, 8-bit step index, reserved
byte, then low nibble first), 505 samples per block, 8 kHz mono 16-bit. Decoded
output matched FFmpeg's IMA WAV decoder byte for byte.

Codec types handled by the app (only 22 observed on this camera):

| Type | Codec |
|---|---|
| 21 | PCM16 |
| 22 | IMA ADPCM |
| 24 | AAC |
| 25 | ADPCM EX |
| 26 | G.711 A-law |

The camera may also embed SEI user data with the marker `macro video -` and a
JSON `ai_array_v1` array of `"%d&%d&%d&%d"` rectangles (static; not observed on
this camera, semantics unknown).

### Talkback (login validated, audio output unverified)

Open a dedicated TCP connection to port 8800 and send a 256-byte login: u32
command 377 at 0, device identifier at 4, current ticket at 8, play channel at
12. A 16-byte reply with command 477 and result 1001 means accepted.

Encode audio as 256-byte IMA ADPCM blocks (type 22), encrypt each frame as a
whole with the live media key (complete 16-byte blocks), then split it into
chunks of at most 496 bytes, each with a 16-byte header:

| Offset | Size | Meaning |
|---|---|---|
| 0 | u32 | Command 180 |
| 4 | u8 | Total fragments |
| 5 | u8 | Fragment index, from 0 |
| 6 | u8 | Codec type |
| 13 | u16 | Fragment payload length |
| 15 | u8 | Frame counter, wraps at 256 |
| 16+ | | Encrypted payload |

Keepalive: command 188 followed by 12 zero bytes, every 3 seconds. Stop by
closing the socket; no stop command was found.

## Configuration reads (validated)

On a new port-8800 connection, using a ticket from a fresh login, send a 256-byte
request: u32 command at 0, u32 device identifier at 4, byte 1 at 8, u32 ticket at
9 (unaligned).

| Request | Response | Content |
|---|---|---|
| 365 | 465 | All configuration: u32 command, u32 result, u16 schema version at 8, u16 section count at 10, then sections with a 5-byte prefix (flag, index, type, u16 length) |
| 321 | 421 | Alarm configuration, 256 bytes |
| 313 | 413 | Recording / SD configuration, 256 bytes |

Result 1001 means success. Selected fields:

- **413**: u32 auto-record code at 9, alarm-record code at 13 (1100 = on), full-disk
  action at 17, capacity at 21, remaining at 25, resolution flags 29–35,
  frame-size code u16 at 37, audio code u16 at 39, **signed i16 SD status at 41**
  (-11 no card, -12 write error).
- **421**: u32 fields at 9/13/17 (voice config / main / sound), 33 alarm
  capability, 37 master alarm, 41 motion, 45 PIR, 49 smoke (1000 = enabled);
  byte 53 time/area capability, 54 schedule count, 55–75 three 7-byte
  schedules, 76/77 grid rows/columns, region grid from 78 (6 × 8 observed).

Sections of 465 use different offsets from the dedicated 413/421 responses.

## Live-session controls (static, not exercised)

16-byte packets on the stream connection, u32 command at 0:

| Command | Layout |
|---|---|
| 199 motion tracking | u32 action @4, u32 value @8, u8 channel @12 |
| 198 sensitivity | u32 action 0 @4, u32 value @8 |
| 196 light | u32 action @4, u8 intensity @8, u8 save @9 |
| 197 night vision / starlight | u32 action @4, u8 save @8, u32 starlight action @9 (unaligned) |
| 190 image orientation | u32 orientation @4 |
| 171 PTZ preset | u32 action @4, u32 preset ID @8, u8 channel @12 |
| 195 zoom | u32 action @4, u32 step @8 |
| 185 touch alarm | u32 action 1001 on / 1002 off @4, u32 value @8 |

SDK action constants: light/night vision on 1001, off 1002, auto 1003;
starlight none 0, on 1001, off 1002; orientation reverse 1000, up 1001, down
1002, left 1003, right 1004; zoom enlarge 1000, narrow 1001, stop 1100. Their
physical effect on this camera is untested.

Legacy one-key arm/disarm (static): command 344, 256 bytes, device ID @4, byte
1 @8, ticket @9, u32 1000 arm / 1001 disarm @13; reply 444 with result @4.

## SD-card listing and replay (static)

This camera reported no SD card, so none of this was exercised.

- **Search 361**, 128 bytes: device ID @4, ticket @8 (no byte-1 prefix), two
  selector bytes @12/13, start date @14 and end date @34 (six u16 each: year,
  month (1-based), day, hour, minute, second), constant 1 @58. Reply 461 (32
  bytes): i16 result @4 (1 ready, -1 none, -2 no card, -3 card error, -4 bad
  device ID, -5 bad ticket).
- **Results 362 → 462**: send 32-byte 362, read 512-byte packets: u16 type @4
  (11 continuation, 12 final), i16 count @6, then 13-byte entries from 8 (i32
  segment ID, u8 type, i32 start seconds, i32 end seconds); at most 38 fit.
- **Replay 363 → 463**, 64 bytes: device ID @4, ticket @8, segment ID @12, u16
  year @16, month/day/hour/minute/second @18–22, file type @23. Reply: i32 result
  @4 (success 1000), i16 width @14, height @16, u32 audio frequency @19.
  Recorded frames have a 22-byte inner header (frame ID, type, width, height,
  u64 timestamp, u32 payload length). Replay framing and decryption are not
  fully recovered.

## Not covered

- The newer "IoT platform" LAN transport on port 8089 (16-byte header + JSON
  methods); present in the app, not validated on this camera.
- Camera-originated alarm events: no LAN event subscription was found. Alarm
  history and push notifications go through the vendor cloud.
- Wi-Fi provisioning / first pairing.

## Constants

The protocol uses three fixed values embedded in the V380 Pro app:
`APP_KEY` (16-byte AES key for the password envelope), and `MEDIA_KEY_A`
(u64) / `MEDIA_KEY_B` (u32) for the media key. They are identical for every
user and can be read from the app's `libhsMediaLibrary.so` (login construction
and media key initialisation, see Evidence).

## Evidence

ARM64 libraries from the V380 Pro 2.2.70 APK (ELF virtual addresses):

| Library | SHA-256 | Functions |
|---|---|---|
| `libhsMediaLibrary.so` | `f94cfbc4a151f0b174f30ba75da02735280a6b0ab371e544e7e5bb787620b338` | LAN login `HSLiveDataV2Transmitter::getDataFromDevice` 0x441b8; PTZ 0x43450–0x4349c; live controls `actionCtrlThreadFunc` 0x427ec; media decryption selection 0x4975c / 0x499a0; talkback `sendSpeakAudioToDevice`; SD replay `HSReplayDataTFV2Transmitter` 0x5ce5c, 0x5ef04 |
| `libaes.so` | `84288a4c6966ab8af04b1658442e4327887649ebc2964e303b685464e0973bfe` | Media decryption 0x3ffc; first-2-KiB routine 0x409c |
| `libmideaCodec.so` | `0582b40023583d17964916fc54942fe28d9898688998fb5c3d47a7cbb5a7b4e6` | `createDecoder` 0x23a38 |

Java/smali references: `com.macrovideo.sdk.defines.Defines`,
`DeviceAlarmSetting`, `DeviceRecordSetting`, `RecordFileHelper`.

No firmware image, vendor code, credential or capture is included in this
repository.
