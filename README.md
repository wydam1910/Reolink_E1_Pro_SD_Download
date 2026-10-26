# Listing and downloading SD-card recordings over the Baichuan protocol (port 9000) — Reolink E1 Pro

Tested on: Reolink E1 Pro (hardware IPC_513SD5, firmware v3.0.0.2356_23062004), Wi-Fi, no usable web/CGI API for SD files.
Method: packet capture of the official Reolink desktop client talking to the camera, decrypted offline. Nothing here was taken from camera firmware.
All values below are placeholders; no real credentials, UIDs or addresses are included.

## 1. Framing

Every message starts with magic `f0 de bc 0a`, then little-endian fields:

| offset | size | field |
|---|---|---|
| 0 | 4 | magic `f0debc0a` |
| 4 | 4 | command id |
| 8 | 4 | body length (extension + payload) |
| 12 | 4 | message id (echoed in the reply) |
| 16 | 2 | status: request `0000`; reply 200 (`c800`) = OK |
| 18 | 2 | class |
| 20 | 4 | *only if class is `1464`/`0000`*: payload offset (= length of the encrypted XML "extension"); 0 if the whole body is XML |

Class bytes `14 65` (request) / `14 66` (reply) use a 20-byte header (no offset field). `14 64` and `00 00` use 24 bytes.

## 2. Login (two steps)

1. Client sends cmd 1, length 0, mid 0, status bytes `12 dc`, class `14 65` (20-byte header, no body).
2. Camera replies cmd 1, class `14 66`, body = XML with `<nonce>`. This body is "BC-obfuscated" (see §3).
3. Client sends cmd 1 with a LoginUser XML where
   `userName = upper(md5(user + nonce))[:31]`, `password = upper(md5(password + nonce))[:31]`,
   body obfuscated with offset 0, class `14 64`, mid 0.
4. Reply status 200 ⇒ logged in. All later bodies are AES.

Login body (exact text matters little, but this is what the official client sends):
```
<?xml version="1.0" encoding="UTF-8" ?>
<body>
<LoginUser version="1.1">
<userName>HASH</userName>
<password>HASH</password>
<userVer>1</userVer>
</LoginUser>
<LoginNet version="1.1">
<type>LAN</type>
<udpPort>0</udpPort>
</LoginNet>
</body>
```

## 3. Encryption

* **BC obfuscation** (login only): XOR with key `[1F 2D 3C 4B 5A 69 78 FF]`; for offset `off = mid & 0xff`, key byte *i* = `K[(off+i) % 8] ^ off`.
* **AES** (everything after login): AES-128-CFB, IV = ASCII `0123456789abcdef`, key = first 16 chars of `upper(md5(nonce + "-" + password))[:31]`. Each message is encrypted with a fresh cipher; extension and payload are encrypted separately.

## 4. Commands used for SD playback

All requests: class `14 64`, AES-encrypted XML body, offset 0. Replies use the same cmd and mid.

| cmd | purpose |
|---|---|
| 151 | (optional) abilities, extension-only message; official client sends it first |
| 114 | get device UID (empty request) → `<uid>` |
| 142 | `DayRecords`: which days of a month have recordings. Reply `dayTypeList/dayType/index` (0-based day of month) |
| 14 | open a file search for one day and stream type → reply contains `<handle>` |
| 15 | fetch the file list for that handle → `FileInfo` entries (`name`, `sizeL/sizeH`, `startTime`, `endTime`) |
| 16 | close the search |
| 143 | download one file (see below) |
| 144 | stop download |
| 93 | keep-alive (send about every 8–10 s during long downloads) |

`streamType` is `mainStream` or `subStream`. On the E1 Pro both exist for every file, with the same name and times; the sub stream (640×352, ~6.5 fps) is about 16× smaller and good enough for motion/object detection. File names look like `01YYYYMMDDHHMMSS`; times in the lists are camera local time.

Request 143 (fill `supportSub` with 1 for sub stream, 0 for main):
```
<body><FileInfoList version="1.1"><FileInfo>
<uid>UID</uid><logicChnBitmap>255</logicChnBitmap><channelId>0</channelId>
<supportSub>0|1</supportSub><streamType>mainStream|subStream</streamType>
<startTime>…</startTime><endTime>…</endTime>
</FileInfo></FileInfoList></body>
```
(with start/end taken from the file list, each as `year/month/day/hour/minute/second` elements, one per line.)

Reply stream for cmd 143, same mid:
1. one XML reply with `<sizeL>` total size;
2. many messages whose *extension* is `<binaryData>1</binaryData>` (+ optionally `<encryptLen>N</encryptLen>`); the payload is raw media. **If `encryptLen` is present, only the first N bytes of that payload are AES-encrypted** — decrypt them and leave the rest;
3. an empty message with status `0x012C` (300) marks the end of the file.

## 5. Media container (BCMedia)

The concatenated payloads form a BCMedia stream:
* `1001` / `1002`: info header (size 32 for `1001`; `1002` carries its size at +4).
* `00dc` / `01dc`: video frame (I / P): `magic(4) codec(4,"H264") payload_size(4) extra_header_size(4) microseconds(4) unknown(4)`, then `extra` bytes, then `payload_size` bytes of Annex-B H.264/H.265, padded to 8 bytes.
* `NNwb`: audio (`u16` size at +4), padded to 8 bytes.

Concatenating the video payloads gives a raw H.264 file. `ffmpeg -f h264 -r <fps> -i raw -c copy out.mp4` produces a clean mp4 (use the average fps from the microsecond timestamps: `(frames-1)/((last-first)/1e6)`; the sub stream is variable-rate).

## 6. Caveats

* Tested on one model/firmware only; other cameras may use different status codes (`0xdd02` vs `0xdd12` at login) or skip `encryptLen`.
* Over Wi-Fi the main stream downloads at roughly 0.5–1 MB/s; use the sub stream for analysis and fetch main only for clips you need.
* One session at a time per camera is safest.

A small working implementation (Python, `cryptography` only) exists in a private project; the protocol above is enough to reimplement it.
