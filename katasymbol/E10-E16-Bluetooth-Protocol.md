# Katasymbol E10 and E16 Bluetooth printing protocol

Katasymbol Android **1.4.20** drives both printers over **Bluetooth Classic SPP/RFCOMM**. The E10 uses `T15Print` and the E16 uses `T16Print`. They share command framing, raster encoding, compression and bulk packet layout. Their startup sequence, density argument and write pacing differ.

**Browser feasibility.** An E10 has been printed to directly from Chrome over BLE and Bluetooth Classic in an independent implementation. The E16 has a viable browser path through Web Serial using the SPP protocol from this APK. An E16 print and its BLE service layout still need hardware verification. This investigation used static APK analysis and offline byte-level checks.

## Source and reproducibility

- Input package `com.supvan.katasymbol`, version `1.4.20`, build `75`. XAPK SHA-256 `66d8070190141e7833e95f68575f438c3a292b1187736c4f8e48fae6a9fa3ba6`.
- The [extracted XAPK](xapk-extracted/) contains the base APK, ARMv7 native-library split, xhdpi resource split, APKPure manifest and icon. The base APK contains five DEX files. The printer code discussed here is in `classes.dex`; the two configuration splits contain no DEX files.
- [JADX output](xapk-extracted/decompiled/sources/) was generated with JADX 1.5.7 using `jadx --no-res --no-imports --threads-count 4 --output-dir xapk-extracted/decompiled xapk-extracted/com.supvan.katasymbol.apk`. JADX reported 160 decompilation errors across the base APK. The E10 ordinary print transfer branch was cross-checked against the [fallback instruction listing](xapk-extracted/T15Print-fallback.java). The E10 date-printing path is outside the ordinary image/label path covered here.
- Start at [E10Device](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/device/E10Device.java), [E16Device](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/device/E16Device.java) and [PrintManager](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/globalsingleton/PrintManager.java). Some numeric literals are rendered by JADX as unrelated resource or library constants. For example, `R2.drawable.sf5304` is 4000, and `R2.attr.common_add_selector` is 332.

## Model identification and Bluetooth transport

| APK model | Printer type | Print process | Driver | Example Bluetooth name prefixes |
| --- | ---: | ---: | --- | --- |
| E10 | 15 | 15 | [T15Print](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/print/T15Print.java) | `T0010`, `T0011`, `T0012`, `T0126`, `T0179` (the APK lists more) |
| E16 | 1501 | 16 | [T16Print](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/print/T16Print.java) | `T0053`, `T0054`, `T0055`, `T0105`, `T0106`, `T0107`, `T0122`, `T0123` |

The discovery and auto-connect code routes both models to Classic because `DeviceManager.isBLEDevice()` excludes their types. `PrintManager` also sets both print processes to Classic. `BluetoothUtils` opens a secure RFCOMM socket with SPP UUID `00001101-0000-1000-8000-00805f9b34fb`. The job goes over the socket's binary streams. [FindDeviceActivity](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/ui/printer/activity/FindDeviceActivity.java), [BluetoothAutoConnImpl](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/bluetooth/BluetoothAutoConnImpl.java), [BluetoothUtils](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/bluetooth/BluetoothUtils.java)

The APK includes these GATT layouts for its BLE-routed devices. Characteristic short IDs in the table expand using the normal Bluetooth base UUID ending in `00805f9b34fb`. The service UUIDs are written out to preserve the APK's exact suffixes.

| Service UUID in APK | Notify | Write |
| --- | --- | --- |
| `0000e0ff-3c17-d293-8e48-14fe2e4da212` | `ffe1` | `ffe9` |
| `0000fee7-0000-1000-8000-0805f9b34fb` | `fec1` | `fec1` |
| `0000ff00-0000-1000-8000-0805f9b34fb` | `ff01` | `ff02` |

`BLEUtils` requests MTU 200 and enables notifications using descriptor `2902`. Independently, one E10 was seen advertising short UUID `FEE7` while exposing the active `e0ff` service, and another E10 was printed to in Chrome using `e0ff` / `ffe9` / `ffe1`. Advertising data alone is insufficient to choose the service. E16 GATT UUIDs and acknowledgement behavior require a device capture. [APK BLEUtils](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/bluetooth/BLEUtils.java), [E10 BLE observation](https://github.com/johnboiles/ble-label-printers#supvan-e10-1), [E10 browser hardware results](https://github.com/khromov/supvan-katasymbol-printer-web-ui)

## Command and reply frames

All offsets are zero-based. Multi-byte numeric fields below are unsigned little-endian.

```text
Host command (16 bytes)
Offset  Bytes        Meaning
0..1    7E 5A        Frame magic
2..3    0C 00        Frame size after the first four bytes
4..6    10 01 AA     Command header
7       opcode
8..9    sum16        Sum of the six bytes at offsets 10 through 15
10..11  00 01        Fixed
12..13  param1
14..15  param2       Zero on ordinary commands, frame count on 0x5C
```

A command reply echoes its opcode at byte 7. The documented family reply prefix is `7E 5A <len LE16> 10 03 55 <opcode>`, with total framed length `4 + len`. Collect serial reads or BLE notifications into a buffer before interpreting replies. Require a matching opcode and enough bytes for the fields being read. The Classic Android path waits roughly 2 seconds for input on a read, and its data-frame path simply reads into an 8-byte buffer without validating the data ACK contents. Exact E16 reply lengths and the data ACK bytes should be captured on hardware. [APK BasePrint](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/print/BasePrint.java), [Family reply framing and parser](https://github.com/khromov/supvan-katasymbol-printer-web-ui/blob/main/src/lib/printer/families/t5080-bt.ts)

| Hex opcode | Meaning | Argument or response fields |
| --- | --- | --- |
| `11` | Query status | Parameter 0. Status at offsets 14–17, print counter at 18–19. |
| `12` | Check device | Parameter 0. E16 sends it before its preflight status loop. |
| `13` | Start print | Parameter 0 for continuous tape. For die-cut stock the app passes its configured out-paper value. |
| `14` | Stop print | Parameter 0. Used to cancel or clear an active print. |
| `16` | Read device name | Parameter 0. String starts at byte 22 and is `len - 18` bytes long. |
| `18` | Trigger material read | Parameter 0. The material-reading methods invoke it. |
| `30` | Return material info | Parameter 0. UUID at 22–28, material code at 29–36, stock serial at 37–38, type at 39, size at 40–41, gap at 42. The same parser fields appear in both print classes. |
| `5C` | Announce one compressed buffer | `param1 = 512`, `param2 = ceil(compressedBytes / 500)`. |
| `BA` | Move tape backward | Parameter is the calculated number of raster columns. |
| `C9` | Set head rate | E16 uses `trunc((((density - 1) / 10) + 0.8) * 100)` in Java float arithmetic. E10 uses the same formula except that names containing `G` or beginning with `T0047B`, `T0052B`, `T0053B`, `T0054B` or `T0122B` use the raw density level. Density range 1–7, driver default 4. |
| `10` | Commit compressed buffer (BUF_FULL) | Parameter 0 for both E10 and E16. |

The `11` response supplies the main print-loop conditions below. Reject short status frames before reading registers. [APK PRINTER_FLAG](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/print/flags/PRINTER_FLAG.java)

| Reply offset | Mask | Meaning used by the app |
| ---: | ---: | --- |
| 14 | `01` | Print buffer occupied |
| 14 | `02` | Label read/write error |
| 14 | `04` | Label stock exhausted |
| 14 | `10` | Material absent / empty |
| 15 | `04` | Device busy |
| 16 | `08` | Cover open |
| 16 | `40` | Printing state active |
| 17 | `01` | Label stock improperly installed |

## Raster and compression

Text and images go through the same bitmap path. The drivers use **8 dots/mm** along the feed and **96 dots across the head** (12 mm at 203.2 DPI). The Android app scales artwork to 88 dots high and centers it across the head, leaving four dots on each edge before position adjustments. For continuous stock the raster width is eight columns per millimetre of label length. In the normal orientation, a dark pixel at feed column `x`, cross-head row `y` sets bit `1 << (y % 8)` in `raster[12*x + floor(y/8)]`. The APK uses a weighted grayscale threshold with a driver default of 125. [APK ImgConverter](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/print/flags/ImgConverter.java)

Split the raster into groups of up to **332 columns**. Encode each group in a **4000-byte** buffer. The app zeroes the buffer at job start and reuses it, so bytes beyond the declared column count can retain earlier values. A one-buffer job starts with a zero-filled tail.

| Buffer offset | Field |
| --- | --- |
| 0–1 | Checksum. Add bytes 2 through 13 and the byte at offset `256*k - 1` for each `k` from 1 through `floor((14 + 12*columns)/256)`. Store the low 16 bits. |
| 2 | Page flags. Start `0x02`, page end `0x04`, job end `0x08`, cut mode `(cut & 7) << 4`, save-paper flag `<< 7`. |
| 3 | `(firstCut & 3) OR (density << 2) OR (1 << 4)`. The APK sets the material field to 1. |
| 4–5 | Number of raster columns in this buffer. |
| 6–7 | `0C 00` (12 bytes per column). |
| 8–11 | `01 00 01 00`. |
| 12–13 | Lead-in columns skipped, followed by zero. Only the first buffer of the job skips columns. |
| 14 onward | Raster bytes, then unused tail up to byte 3999. |

For continuous tape the app looks for ink in the first 48 columns, skips up to one column before the first ink, and uses 47 as the skip when that region is blank. If the skip is below 47, the app moves tape back by `48 - skip` columns before sending data. Otherwise it sends `BA` with parameter zero. For die-cut stock the search limit is based on `trunc((6.5 - gapMm)*8)`. Keep the initial calibration label longer than the search window. [APK PAGE_REG_BITS](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/print/flags/PAGE_REG_BITS.java)

Compress the **whole 4000-byte buffer** as its own LZMA-Alone stream. The settings are dictionary **8192**, `lc=3`, `lp=0`, `pb=2`, 128 fast bytes, encoder match-finder setting 1, end marker disabled. The 13-byte header is `5D 00 20 00 00 A0 0F 00 00 00 00 00 00`: five property bytes followed by the eight-byte uncompressed size. [APK LzmaUtils](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/print/compression/LzmaUtils.java), [APK LZMA Encoder](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/print/compression/LZMA/Encoder.java)

## Bulk frame and print sequence

Announce each compressed buffer using `5C(512, N)`, where `N` is the number of frames for that buffer. Then send `N` frames of exactly 512 bytes:

```text
Offset   Bytes                Meaning
0..5     7E 5A FC 01 10 02    Bulk frame header, length field = 508
6..7     AA BB                Data marker
8..9     sum16                Sum of unsigned bytes 10 through 511
10       index                Zero-based within this compressed buffer
11       count                Total frames in this compressed buffer
12..511  payload              500-byte field
```

The first frame buffer starts zero-filled. For a final short frame in a multi-frame buffer, the Android app overwrites only the used part of that reused 500-byte field. Its checksum includes the entire field, including any remaining bytes from the preceding frame. On SPP, allow about 2 seconds for the app-level data ACK after each full 512-byte frame. The E16 always splits each frame into four 128-byte writes. The E10 does so when the stored Bluetooth name contains uppercase `E`. Otherwise it writes the full frame at once. The split helper sleeps 50 ms before each piece and the Classic write helper adds a 10 ms pause after its writes. Both drivers wait another 50 ms after the last data frame before sending `10(0)`. [APK BasePrint](xapk-extracted/decompiled/sources/com/fhit/app_iprinter/communication/print/BasePrint.java), [E10 transfer fallback listing](xapk-extracted/T15Print-fallback.java)

| Stage | E10 | E16 |
| --- | --- | --- |
| Preflight | Poll `11(0)` at 100 ms intervals until idle, then send `C9`. | Send `C9`, then `12(0)`. Poll `11(0)` at 100 ms intervals. If a print is still active, the APK sends `14(0)` during this loop. Check cover and stock errors. |
| Start | Send `13(outPaper)`, poll `11` until printing state is active, then send `BA(backColumns)`. | Send `13(outPaper)`, then `BA(backColumns)` once the raster is prepared. |
| Per buffer | Wait for `11` to show buffer clear and no blocking error. Send `5C`, the data frames with per-frame ACKs, then `10(0)`. | Same. |
| Finish | Poll `11` for printing and busy flags to clear. | Same. |

For a browser implementation, serialize requests and use bounded waits. A useful completion signal is a final status reply with printing and busy clear. If a state-changing write times out or Bluetooth disconnects mid-job, report the result as unknown and require the user to start a new job explicitly.

## Offline test vector

This synthetic fixture models a 32 mm continuous label with one dark pixel at feed column 48, cross-head row 4, density 4 and cut mode 0. Skip 47 columns and set `BA` to 0. The encoded buffer has 209 columns. Set byte 26 to `10`, write the 14-byte header below, and leave all other bytes zero. The reference LZMA stream was generated using liblzma with no end marker and decoded back to the original 4000 bytes. Compression implementations can produce different valid streams.

```text
Status query  7e 5a 0c 00 10 01 aa 11 01 00 00 01 00 00 00 00
Check device  7e 5a 0c 00 10 01 aa 12 01 00 00 01 00 00 00 00
Head rate 110 7e 5a 0c 00 10 01 aa c9 6f 00 00 01 6e 00 00 00
Start print   7e 5a 0c 00 10 01 aa 13 01 00 00 01 00 00 00 00
Paper back 0  7e 5a 0c 00 10 01 aa ba 01 00 00 01 00 00 00 00
Announce 1    7e 5a 0c 00 10 01 aa 5c 04 00 00 01 00 02 01 00
Commit buffer 7e 5a 0c 00 10 01 aa 10 01 00 00 01 00 00 00 00

Raw page header
2c 01 0e 10 d1 00 0c 00 01 00 01 00 2f 00

LZMA stream (55 bytes)
5d 00 20 00 00 a0 0f 00 00 00 00 00 00 00 16 00 3d ce
c6 84 8c ac 57 58 fc ab cf 3f 44 2a d9 a7 a4 37 82 b8
13 e5 28 f0 3a b2 df 02 53 74 7e cf 4c 16 37 e3 fd eb 00

512-byte data frame
7e 5a fc 01 10 02 aa bb 8c 15 00 01
followed by the 55-byte LZMA stream and 445 zero bytes
```

## What a simple webpage would need

1. **Connect.** Use an HTTPS page or a local development origin and a user-initiated Connect button. For the APK-compatible path, pair the printer in OS settings, then use `navigator.serial.requestPort()` and open the SPP port. Chrome supports Bluetooth RFCOMM via Web Serial on desktop from Chrome 117 and also supports it on Android. Availability still depends on the OS exposing the paired printer's SPP service. A tested E10 project reports successful Web Serial printing on Android and unreliable SPP port opening on macOS. [Chrome desktop SPP documentation](https://developer.chrome.com/blog/serial-over-bluetooth/), [Chromium Android Web Serial announcement](https://groups.google.com/a/chromium.org/g/blink-dev/c/yGhvQ6mEmcY/m/xr-aYJ-_AQAJ), [E10 test results](https://github.com/khromov/supvan-katasymbol-printer-web-ui)
2. **Support E10 BLE where available.** Request a device by name prefix and grant `e0ff` as an optional service. Subscribe to `ffe1` before writing to `ffe9`. Start with the write sizes and acknowledgement handling from the browser implementation that was tested on an E10. A resolved GATT write only confirms the transport operation, while the printer's notification acknowledges the protocol frame. Verify the E16's actual GATT services before enabling E16 BLE in a page. [Chrome Web Bluetooth guide](https://developer.chrome.com/docs/capabilities/bluetooth), [Tested E10 implementation](https://github.com/khromov/supvan-katasymbol-printer-web-ui/blob/main/src/lib/printer/families/t15.ts)
3. **Render and print.** Draw text or a locally selected image onto a white Canvas, show a monochrome preview, and use the raster, LZMA and packet layout above. JavaScript or WebAssembly can do the encoding entirely in the browser. Implement the matching E10 or E16 state machine after identifying the printer. A simple first test is a roughly 32 mm continuous-tape label with an asymmetric black marker to reveal orientation, clipping or lead-in errors.
4. **Verify on hardware.** First send the status query and log the reply. Then send one short label and record command replies, the data ACK, final status and physical output. For the E16, SPP is the first path to test because it is the transport selected by the APK.
