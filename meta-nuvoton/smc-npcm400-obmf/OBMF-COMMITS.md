# OBMF-ICP 1.0.0 over USB: OpenBMC and OpenBIC commit guide

This guide maps every OBMF commit on the BMC (Linux driver) to the matching commit on SMC1 (Zephyr firmware).
Use it to decide which commits must be deployed together and which can be taken alone.

## 1. Where the code is

| Role | Repository | Branch | Commit range |
|---|---|---|---|
| SMC1 firmware (device) | [maxdog988/OpenBIC](https://github.com/maxdog988/OpenBIC) | `npcm-v2.6-dev` | `ad048739cb..40fa8f3e00` (49 commits, plus the commit that adds this guide and the zephyr patches) |
| SMC1 platform (USB class driver, I3C driver) | [maxdog988/zephyr](https://github.com/maxdog988/zephyr) | `2.6-dev` (local `zephyr_nuvoton`) | `0f9a650352..b091b68916` (11 commits) |
| BMC kernel driver (host) | [maxdog988/linux](https://github.com/maxdog988/linux) | `6.12-OBMF-DEV` | `a33374518f..14276b81cc` (40 commits) |

Driver path on the BMC: `drivers/usb/class/obmf/`. SMC1 path: `meta-nuvoton/common/service/obmf/`.

SMC1 builds on the `zephyr_nuvoton` tree, which holds the OBMF USB class driver (`subsys/usb/class/obmf.c`) and the NPCM4xx I3C driver. Its 11 OBMF-related commits are listed in section 4 (USB class and platform). They run only on SMC1 and change no byte on the wire, so none of them must be paired with a BMC commit. Two OpenBIC commits depend on them at build time (section 3.4).

**Push state:** the last two OpenBIC commits (`2b1cbf93de`, `40fa8f3e00`) and the last three zephyr commits (`67560c4650`, `f1760d5dfb`, `b091b68916`) are local and not pushed yet, so their links return 404 until they are. Both the OpenBIC and the `zephyr_nuvoton` history were rewritten earlier, so pushing needs a force push. The BMC branch on GitHub matches this guide (40 commits, head `14276b81cc`).

## 2. How to read this guide

| Mark | Meaning |
|---|---|
| 🔴 `H*` | **Hard pair.** The SMC1 and BMC commits with the same ID must be deployed together. A mixed pair breaks the protocol or the data. |
| 🟡 `S*` | **Soft pair.** A mixed pair still works but reports different errors or has reduced robustness. |
| 🟢 independent | The commit works on its own with either version of the other side. |

The `Spec` column is the section named in the commit's `Reference:` line (OCP OBMF-ICP Spec 1.0.0).

## 3. Deployment pairs

### 3.1 Hard pairs (deploy together)

| ID | Topic | SMC1 commit(s) | BMC commit(s) | Why |
|---|---|---|---|---|
| **H1** | MMIO message layout | [`1e96`](https://github.com/maxdog988/OpenBIC/commit/1e96f928282200c2e3bde50526cd271c3ba17d78), [`09bc`](https://github.com/maxdog988/OpenBIC/commit/09bc69e6ec5b806033f044814252cc174d918024) | [`8026`](https://github.com/maxdog988/linux/commit/d9d578d45c1248ecbf60787a56f77208e9a154d9) | Both ends parse the same bytes, so the format must change on both. The new MMIO message has a 2-byte sub-header (Transaction, Reserved) and no Write Length field (Section 5.3, Tables 5, 10 and 14). SMC1 builds and parses Channel 0 requests, and the BMC builds them and parses the replies. If only one side changes, the other reads every address and length at the wrong offset, so discovery fails and writes are rejected. |
| **H2** | Channel readiness | [`68af`](https://github.com/maxdog988/OpenBIC/commit/68af436a7178f25a8aa9745798bd8427197a9e25) | [`799b`](https://github.com/maxdog988/linux/commit/799b41aab79f), [`20d2`](https://github.com/maxdog988/linux/commit/20d20da33f5b) | Channel readiness is a handshake between the two sides. The Host enables a channel through CHANNEL_CONTROL before it uses it (Section 4.3 step 2.b), and the Device must refuse requests until then (Section 5.15, Table 65). The SMC1 fix answers a disabled channel with 0x03 Channel not ready. The BMC fix is what sends the enable first and waits to be enabled. A new SMC1 with an old BMC sends 0x03 for requests that the old BMC issues too early, and an old SMC1 does not enforce the handshake. |
| **H3** | Header-only replies | [`96d7`](https://github.com/maxdog988/OpenBIC/commit/96d7f201edcd9d1b7ea9011edc14e7fab898a551) | [`471a`](https://github.com/maxdog988/linux/commit/471ab1c87519), [`20d2`](https://github.com/maxdog988/linux/commit/20d20da33f5b) | The two sides disagree on what a reply with no payload looks like. Replies such as 0x01, 0x03 and 0x05 carry only the 4-byte header (Section 5.1, Table 3, Size may be 0). The SMC1 fix makes it transmit them. The BMC fix makes it accept and send them as a complete message. With only the BMC fix, the SMC1 never sends these replies and the BMC waits until its timeout. With only the SMC1 fix, an old BMC does not recognise a header-only message. |
| **H4** | GPIO Low/High encoding | [`b913`](https://github.com/maxdog988/OpenBIC/commit/b913cba37f7769dba393e5aec4aa9681b4e1d284) | [`be1c`](https://github.com/maxdog988/linux/commit/be1c8757c976f434becd487d44e0ce727e80fa22) | Low and High are values written into messages (Section 5.4). The SMC1 encodes the value of a line and the BMC decodes it, so both must agree on Low=0 and High=1. If only one side swaps, every output is driven to the opposite level and every input is read inverted. |
| **H5** | GPIO interrupt config | [`9f8e`](https://github.com/maxdog988/OpenBIC/commit/9f8ef92de9bc7f4d6537063285f4d43ee2c5580e) | [`b7ab`](https://github.com/maxdog988/linux/commit/b7ab97f60904522474babd795ba683bf7114136d) | Set Interrupt Config is a request from the BMC that SMC1 must accept. The BMC fix stops marking the interrupt as enabled when the device rejects the request. The SMC1 fix is the one that accepts the request. An old SMC1 rejects it with 0x01, so the new BMC leaves `irq_enabled` clear and ignores all notifications. Deploy SMC1 first or together. |
| **H6** | I/O channel header | [`f9ba`](https://github.com/maxdog988/OpenBIC/commit/f9baa1a6f1fd05984382cd2beaf485695410bb3d) | [`b85a`](https://github.com/maxdog988/linux/commit/90e474d11f963b75aa6b5a61cada4577ca70cb6d) | The I/O message header is shared by both sides. It shrinks from 2 bytes to 1 byte (Section 5.11, Table 52), and the port address follows it. If only one side changes, the port address starts one byte too early or too late, so every I/O access hits a wrong port. |
| **H7** | I/O read response | [`3fe1`](https://github.com/maxdog988/OpenBIC/commit/3fe18c068e93650bf82c83ebe8a1164d246b4fa4) | [`5c59`](https://github.com/maxdog988/linux/commit/7aa0469af07a48188ff539edb86ddbeb6a9daa26) | The I/O read response layout is fixed by both sides. The reply payload is the data only (Section 5.11.3, Table 56), and the status is carried in the message header (Section 5.1, Table 3). An old SMC1 still puts a status byte in front of the data, so the new BMC sees one byte too many and returns -EIO for every read. A new SMC1 with an old BMC loses the first data byte. |
| **H8** | Mem-dev read payload | [`461d`](https://github.com/maxdog988/OpenBIC/commit/461da2dc0c3fedd9510781cb8e9a23090cbce2ba) | [`43c0`](https://github.com/maxdog988/linux/commit/79c4d38e1f5652e70a2beea069ebd4ec35bbaab2) | The Memory Device read payload layout is also shared. Both sides drop the status byte at the start of the payload, because the status is already in the header (Section 5.12.2, Table 62). If only one side changes, every read returns data shifted by one byte, which silently corrupts the data instead of failing. |
| **H9** | I3C IBI target array | [`0c61`](https://github.com/maxdog988/OpenBIC/commit/0c6123748831b6f65cdddbd2588536798042fe3b) | [`9ef6`](https://github.com/maxdog988/linux/commit/d95e32cf2bed760ce1a6c3b92aaeeb9e39fe5bf2) | SMC1 writes the I3C Configuration Data and the BMC reads it. The IBI target count is at offset 0x0E and the array starts at 0x0F with 6-byte entries (Section 5.15.2, Table 81). If the two sides disagree, the BMC reads the target array from the wrong offset and registers wrong or no IBI targets. |

### 3.2 Soft pairs

| ID | Topic | SMC1 commit(s) | BMC commit(s) | Effect of a mixed pair |
|---|---|---|---|---|
| **S1** | Status code mapping | [`7347`](https://github.com/maxdog988/OpenBIC/commit/73479efe44059d25ec11819f81c2d472aba1a8ea), [`3670`](https://github.com/maxdog988/OpenBIC/commit/36700c9c4c356b2b8eac240d5a9c3030187708e1), [`2526`](https://github.com/maxdog988/OpenBIC/commit/25264f1b8a6f3b1c786c62f6fbbcc86653c98cba), [`45db`](https://github.com/maxdog988/OpenBIC/commit/45dbd60e2afd0e316c9eebc4f909e26b8174daf9), [`1095`](https://github.com/maxdog988/OpenBIC/commit/10954ef65d82fbb3611497aa1135a5b65f06ec4b), [`496e`](https://github.com/maxdog988/OpenBIC/commit/496e1295e4835ee02494902b13377d403b4641f2) | [`27cb`](https://github.com/maxdog988/linux/commit/27cb8215007f), [`2522`](https://github.com/maxdog988/linux/commit/25222d68ecac676d947ab834badd5d16cd7e1c40), [`d956`](https://github.com/maxdog988/linux/commit/d95653374cb93918be03cb7794ab6c883407ecb9), [`12bf`](https://github.com/maxdog988/linux/commit/39cd5242b0fb2aa84e67cffd47836c4f247b173a), [`9868`](https://github.com/maxdog988/linux/commit/7e5e64c5ad385b8967a57e3f5ac1cdfb1be47b05), [`20cd`](https://github.com/maxdog988/linux/commit/5d781661845c1c737a7f24c7f292112e319fb758), [`9b52`](https://github.com/maxdog988/linux/commit/4022017e68c0f43e8fcc7cb7fd054dd691c57a78) | Status codes are chosen by the SMC1 and interpreted by the BMC (Section 5.2, Table 4, and the channel-specific tables). The SMC1 commits stop it from sending reserved or wrong codes and use the right code per condition. The BMC commits map each code to the right errno. Each side is correct alone, so a mixed pair works. Only the error that the BMC reports is less precise, for example -EIO instead of -EOPNOTSUPP or -EBUSY. |
| **S2** | Serial 253-byte limit | [`cb43`](https://github.com/maxdog988/OpenBIC/commit/cb430acf4ce3f74cc4c3e4def62118c7e9510aa4) | [`c2da`](https://github.com/maxdog988/linux/commit/c2daf0fe1f374f51f99ebc2e876f066146f232d4) | The 253-byte serial limit exists on both sides because each one builds or receives the request. The SMC1 refuses a request longer than 256-3 bytes. The BMC splits writes so that none exceeds it. A mixed pair still transfers data. An old BMC may build requests that SMC1 rejects, and an old SMC1 may accept more than the BMC will send, so the limit is only enforced on one end. |
| **S3** | Payload size limits | [`929f`](https://github.com/maxdog988/OpenBIC/commit/929f51f9f61a8ad39ea1100128d10d7234b31919), [`52cc`](https://github.com/maxdog988/OpenBIC/commit/52cc0e3b96239d682c89fedbdf129e795a5b62b7), [`93e9`](https://github.com/maxdog988/OpenBIC/commit/93e91072481a0d17663847ad34d376a925320e4c) | [`8103`](https://github.com/maxdog988/linux/commit/530d4c12f80fac36ac1dde54ab107a70150f3114), [`be90`](https://github.com/maxdog988/linux/commit/14276b81ccc8f3994f91f928ae498d89a740ddcc), [`c354`](https://github.com/maxdog988/linux/commit/c354bef1e36e718b9bb0abb1da446484e8ff0dd0), [`579c`](https://github.com/maxdog988/linux/commit/4a819005a5e5b0c3525cb1c6f3552c5744ea3b89) | Payload limits are advertised by the SMC1 (MAX_REQUEST_PAYLOAD_SIZE and MAX_RESPONSE_PAYLOAD_SIZE, Section 5.15, Table 65) and the BMC must respect them when it builds requests. The SMC1 commits make it enforce its own limits and reply 0x06. The BMC commits make it size requests and reads from the advertised values, so it never sends an oversized request. With only one side changed, the larger request is not prevented and just gets an error reply. |
| **S4** | GPIO notification path | [`50ea`](https://github.com/maxdog988/OpenBIC/commit/50ea0b8b124ca16be253ce585e612d82ec1e4a1a), [`9e85`](https://github.com/maxdog988/OpenBIC/commit/9e853a987e1d9f275bb6599a84bde30bf2e73aad) | [`bc1c`](https://github.com/maxdog988/linux/commit/bc1c424afa0399237e138642cd5c1abad5bc2d09) | A GPIO notification travels from SMC1 to the BMC. The SMC1 commits send the notification with the right GPIO_INDEX and only for enabled interrupts. The BMC commit maps the index back to a line and handles the notification. The BMC must have enabled the line first. With one side old, notifications for some lines are dropped or are mapped to the wrong line. |
| **S5** | Stale and late responses | [`aa6d`](https://github.com/maxdog988/OpenBIC/commit/aa6d3a78abfd6cdc7a184f0b15ef026c35c4551f), [`f872`](https://github.com/maxdog988/OpenBIC/commit/f8729b3250492911a4dff54a6743e656e1fec856) | [`be4c`](https://github.com/maxdog988/linux/commit/be4cf5661f58), [`a880`](https://github.com/maxdog988/linux/commit/a880a17bd779), [`8fa2`](https://github.com/maxdog988/linux/commit/8fa2e440ad63) | Each side can receive a response after the request has timed out. The SMC1 commits keep a single outstanding request and time it out. The BMC commits drop late or stale responses and never write them into a caller that already returned. Each side protects itself, so a mixed pair works. Without the fix on a side, that side can match an old response to a new request. |
| **S6** | Unknown-channel replies | [`6d81`](https://github.com/maxdog988/OpenBIC/commit/6d812f052e80d0d4d895983ac9f6b68bb1f0cd60) | [`908c`](https://github.com/maxdog988/linux/commit/908c48bfe39c38f7207b74fbdc61c9c40e8c33c0) | A fragmented request to an unknown channel must get exactly one reply (Section 5.2, Table 4, 0x05; Section 3.2, Table 1 requirement 12). Each side is both a sender and a receiver of such requests, so each needs the fix. A mixed pair still works, but the older side may send one reply per fragment. The peer then sees extra responses that no request is waiting for and drops them. |

### 3.3 Independent commits

Everything not listed above is independent. The main ones:

- **SMC1:** reassembly and request timeout, Long Read/Write wrap check, MMIO range checks and window reporting, GPIO input validation, I2C and I3C responder fixes, SPI read limit, shell commands.
- **BMC:** major-version check, DEVICE_ROLE read, GPIO line-name leak, serial hangup and full-NACK handling, malformed POST-code answers, I/O range check.

### 3.4 SMC1-internal pairs (OpenBIC and zephyr_nuvoton)

These pairs are inside SMC1. None involves a BMC commit.

| ID | Topic | OpenBIC commit | zephyr commit | Why |
|---|---|---|---|---|
| **Z1** | USB backpressure | `2b1cbf93de` | `b091b68916` | The class driver adds an optional `rx_space()` op and `obmf_usb_rx_resume()`. The application reports its free ring space and calls resume after each read. OpenBIC `2b1cbf` does not build without the zephyr commit. The zephyr commit alone is safe, because `rx_space` is optional and a missing op keeps the old behaviour. Together they stop a full ring buffer from dropping bytes mid-message: the Bulk OUT endpoint keeps NAKing and the host controller retries. |
| **Z2** | Per-device MCTP UUID | `40fa8f3e00` | none | Independent of the BMC and zephyr code. It only matters when more than one device shares a BMC (see 7.4): give each SMC its own `-DOBMF_MCTP_UUID_NODE=N`. |

## 4. Commits by channel

History is ordered by channel on both sides: Channel 0, GPIO, Serial, I2C, then MMIO, I3C, SPI, I/O and Memory Device. Within a channel, commits are in apply order.

### Channel 0 (Control)

**SMC1 (OpenBIC)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`1e96f92828`](https://github.com/maxdog988/OpenBIC/commit/1e96f928282200c2e3bde50526cd271c3ba17d78) | use the OBMF-ICP 1.0.0 MMIO message layout | 🔴 H1 | Section 5.15 (Channel 0 is accessed with MMIO Short Read/Write); Section 5.3 (OBMF MMIO Channel, Table 5 - 2-byte header: Transaction in byte 0, byte 1 Reserved, payload at byte 2); Sections 5.3.3 and 5.3.7 (Short/Long Write Request, Tables 10 and 14 - Data follows the Address, no length field) |
| [`73479efe44`](https://github.com/maxdog988/OpenBIC/commit/73479efe44059d25ec11819f81c2d472aba1a8ea) | never send the reserved status code 0x07 | 🟡 S1 | Section 5.2 (OBMF-ICP Status Codes, Table 4 - 0x01 Invalid command or parameter, generic codes end at 0x06, 0x07-0x3F Reserved); Section 5.1, Table 3 (OBMF-ICP Message Format - 4-byte message header) |
| [`0994c413b5`](https://github.com/maxdog988/OpenBIC/commit/0994c413b5d3ede03feb1fc118a57179eb0075c0) | answer requests on consumer and unknown channels | 🟢 independent | Section 3.2, Table 1 requirement 12 (request-response model); Section 5.2, Table 4 (0x01 Invalid command or parameter, 0x05 Unknown OBMF-ICP Channel); Sections 5.7, 5.9 and 5.12 (I2C Target, IPMI and Memory Device Channel Roles, Tables 29, 44 and 59 - the Producer is not a Requester) |
| [`38e0415cfd`](https://github.com/maxdog988/OpenBIC/commit/38e0415cfd2ab6805b1435ccc72ac6bc3d46954c) | implement CHANNEL_CONTROL and CONFIG_STATUS semantics | 🟢 independent | Section 5.15 (Channel 0 reset/boot entry, manufacturing and runtime configuration flows); Table 64 (CONFIG_STATUS - MANUF_CONFIG_LOCK_SUPPORTED bit 0 ... RUNTIME_CONFIG_APPLIED bit 4); Table 65 (CHANNEL_STATUS and CHANNEL_CONTROL - CHANNEL_ENABLE stored, RESET/APPLY/LOCK_REQUEST/CLEAR_ERROR write-1 self-clearing) |
| [`10954ef65d`](https://github.com/maxdog988/OpenBIC/commit/10954ef65d82fbb3611497aa1135a5b65f06ec4b) | report oversized transfers as "Request size not supported" | 🟡 S1 | Section 5.2, Table 4 (0x06 Request size not supported); Section 5.3, Table 7 (0x40 Address out-of-range); Section 5.15 and Section 3.2, Table 1 requirement 30 (64-byte minimum Short Read/Write payload for Channel 0) |
| [`496e1295e4`](https://github.com/maxdog988/OpenBIC/commit/496e1295e4835ee02494902b13377d403b4641f2) | reject requests larger than MAX_REQUEST_PAYLOAD_SIZE | 🟡 S1 | Section 5.15, Table 65 (MAX_REQUEST_PAYLOAD_SIZE - maximum size of a Request for this channel); Section 5.2, Table 4 (0x06 Request size not supported); Section 3.2, Table 1 requirement 26 (Device-specific message size) |
| [`ff4a5333a8`](https://github.com/maxdog988/OpenBIC/commit/ff4a5333a8525442589bcc11299dede2736d6ea3) | reject Long Read/Write addresses that wrap the bounds check | 🟢 independent | Sections 5.3.5 and 5.3.7 (Long Read/Write Request, Tables 12 and 14 - 64-bit Address); Section 5.3, Table 7 (MMIO Channel-specific Status Codes - 0x40 Address out-of-range); Section 5.15 (Channel 0 uses the MMIO access methods); Section 6.5 (Channel Isolation - one channel must not compromise others) |
| [`f8729b3250`](https://github.com/maxdog988/OpenBIC/commit/f8729b3250492911a4dff54a6743e656e1fec856) | keep one outstanding request per channel and time requests out | 🟡 S5 | Section 3.2, Table 1 requirement 15 (only one outstanding request per communication channel); Section 6.6 (Transport Guarantees - OBMF-ICP defines no timeouts; upper layers implement them to handle an endpoint that does not respond) |
| [`146add8695`](https://github.com/maxdog988/OpenBIC/commit/146add8695c5636b4f90103332ffe3aa95fe1412) | enforce the access type of every configuration field | 🟢 independent | Section 5.15, Table 63 (Channel 0 Tables Convention - "r" read-only, "rw" read-write) and Table 65 (Channel Configuration Space Common Header - CHANNEL_NAME rw, CONFIG_SIZE); Section 5.15.2, Tables 75-87 (per-channel Configuration Data access types); Section 5.3, Table 7 (0x40 Address out-of-range, 0x41 Access denied for privilege level) |
| [`96d7f201ed`](https://github.com/maxdog988/OpenBIC/commit/96d7f201edcd9d1b7ea9011edc14e7fab898a551) | send messages that have no payload | 🔴 H3 | Section 5.1, Table 3 (Size - the Payload may be empty, e.g. Tables 11 and 15 "No Payload"); Section 5.2, Table 4 (status carried in the message header); Section 3.2, Table 1 requirement 12 (request-response model - every request is answered) |
| [`68af436a71`](https://github.com/maxdog988/OpenBIC/commit/68af436a7178f25a8aa9745798bd8427197a9e25) | answer requests on a disabled channel with "Channel not ready" | 🔴 H2 | Section 5.15, Table 65 (CHANNEL_CONTROL [0] CHANNEL_ENABLE; CHANNEL_STATUS HEALTH_STATUS 2 - Channel Disabled); Section 4.3, system operation sequence step 2.b (the Host enables the channels before it starts using them); Section 5.2, Table 4 (0x03 Channel not ready / busy) |
| [`ecb6b35a2a`](https://github.com/maxdog988/OpenBIC/commit/ecb6b35a2a6126583acbc4a2caba12e1071b476a) | share OBMF_RSP_BUF_SIZE with the channel handlers | 🟢 independent | Section 5.15, Table 65 (MAX_REQUEST_PAYLOAD_SIZE / MAX_RESPONSE_PAYLOAD_SIZE); Section 3.2, Table 1 requirement 26 (Device-specific message size) |
| [`6e936fe368`](https://github.com/maxdog988/OpenBIC/commit/6e936fe368374b038966b18a7ada2c5aff96b5e7) | end a received message by its Size field | 🟢 independent | Section 5.1 (Common Message Header, Table 3, Size field) |
| [`72d90f93c0`](https://github.com/maxdog988/OpenBIC/commit/72d90f93c0887b9823357cabf89dc517bee1ac99) | harden fragment reassembly and request timeout | 🟢 independent | Section 5.15 (OBMF-ICP Control Channel - Channel 0) |
| [`6d812f052e`](https://github.com/maxdog988/OpenBIC/commit/6d812f052e80d0d4d895983ac9f6b68bb1f0cd60) | reply once to an unknown-channel request and clear discard on reset | 🟡 S6 | Section 5.15, Table 65 (CHANNEL_CONTROL [1] RESET); Section 5.2, Table 4 (0x05 Unknown OBMF-ICP Channel); Section 5.1, Table 3 (Size field - a request is answered once, not per fragment) |

**BMC (linux)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`27cb821500`](https://github.com/maxdog988/linux/commit/27cb8215007f) | map response status codes per channel type | 🟡 S1 | Section 5.2 (OBMF-ICP Status Codes, Table 4 - 0x00-0x3F generic, 0x40-0x7F defined per channel); Section 5.3, Table 7 (MMIO Channel-specific Status Codes); Section 5.11, Table 54 (IO Channel-specific Status Codes); Section 5.15 (Channel 0 is accessed with MMIO transactions) |
| [`338ed07ebd`](https://github.com/maxdog988/linux/commit/338ed07ebd52) | reject devices with an unsupported major version | 🟢 independent | Section 3.2, Table 1 requirement 2 (Host performs version discovery and shall reject a device whose OBMF-ICP version is not supported); Section 5.15, Table 64 (OBMF_VER - VERSION_MINOR_BCD at byte 0, VERSION_MAJOR_BCD at byte 1) |
| [`471ab1c875`](https://github.com/maxdog988/linux/commit/471ab1c87519) | answer requests on unknown channels and Channel 0 | 🔴 H3 | Section 3.2, Table 1 requirement 12 (request-response model) and requirement 28 (Host shall not expose Channel 0); Section 4.2 (Channel 0 is always exposed by the Device, never by the Host); Section 5.2, Table 4 (0x05 Unknown OBMF-ICP Channel / Channel not supported) |
| [`be4cf5661f`](https://github.com/maxdog988/linux/commit/be4cf5661f58) | never copy a late response into a returned caller | 🟡 S5 | Section 6.6 (Transport Guarantees - OBMF-ICP has no timeouts; upper layers implement them for an unresponsive endpoint); Section 3.2, Table 1 requirement 15 (one outstanding request per channel) and requirement 12 (request-response model) |
| [`a880a17bd7`](https://github.com/maxdog988/linux/commit/a880a17bd779) | drop stale responses before and during discovery | 🟡 S5 | Section 5.3, Table 5 (MMIO header - Transaction) and Sections 5.3.2, 5.3.4, 5.3.6, 5.3.8 (Short/Long Read/Write Response, Tables 9, 11, 13, 15 - Data is exactly Read Length bytes on Success, no payload for writes); Section 5.15 (Channel 0 uses MMIO Short Read/Write); Section 3.2, Table 1 requirement 15 (one outstanding request per channel); Section 6.6 (Transport Guarantees - timeouts are implemented above OBMF-ICP) |
| [`799b41aab7`](https://github.com/maxdog988/linux/commit/799b41aab79f) | enable a channel before registering it | 🔴 H2 | Section 4.3, system operation sequence step 2.b (the Host enables the channels, and from that point starts using them); Section 5.15, Table 65 (CHANNEL_CONTROL [0] CHANNEL_ENABLE) |
| [`95f7f7ebd6`](https://github.com/maxdog988/linux/commit/95f7f7ebd699) | read the per-channel DEVICE_ROLE | 🟢 independent | Section 4.3 (Channel 0 discovery) and Table 65 (DEVICE_ROLE) |
| [`20d20da33f`](https://github.com/maxdog988/linux/commit/20d20da33f5b) | serve a channel only once it is registered | 🔴 H3 | Section 5.2, Table 4 (0x03 Channel not ready); Section 4.3, system operation sequence step 2.b; Section 5.15, Table 65 (CHANNEL_CONTROL [0] CHANNEL_ENABLE) |
| [`8fa2e440ad`](https://github.com/maxdog988/linux/commit/8fa2e440ad63) | answer requests queued before the driver probed | 🟡 S5 | Section 3.2, Table 1 requirement 15 (a request is answered); Section 6.6 (Transport Guarantees); Section 5.15 (Channel 0 discovery uses MMIO Short Read) |
| [`908c48bfe3`](https://github.com/maxdog988/linux/commit/908c48bfe39c38f7207b74fbdc61c9c40e8c33c0) | answer a fragmented unknown-channel request once | 🟡 S6 | Section 5.2, Table 4 (0x05 Unknown OBMF-ICP Channel); Section 5.1, Table 3 (Size field spans the fragments of one message); Section 3.2, Table 1 requirement 12 (one response per request) |
| [`b1edeb28ce`](https://github.com/maxdog988/linux/commit/b1edeb28cefb542ac0bf87dd07b52784f8496c0e) | retry the OBMF_VER read when discovery times out | 🟢 independent | Section 4.3 (Channel 0 discovery); Section 5.15, Table 64 (OBMF_VER); Section 6.6 (Transport Guarantees - OBMF-ICP has no timeouts, the upper layer retries) |

### GPIO

**SMC1 (OpenBIC)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`9f8ef92de9`](https://github.com/maxdog988/OpenBIC/commit/9f8ef92de9bc7f4d6537063285f4d43ee2c5580e) | implement Get/Set Interrupt Config | 🔴 H5 | Section 5.4.1 (GPIO Channel Request, Table 18 - commands 2 Get Interrupt Config and 3 Set Interrupt Config, initiated by the Consumer; trigger values 0-5, 6-15 Reserved); Section 5.4, Table 16 (GPIO Channel Roles - Producer is Responder) and Table 17 (0x41 Interrupt not supported for GPIO Index) |
| [`50ea0b8b12`](https://github.com/maxdog988/OpenBIC/commit/50ea0b8b124ca16be253ce585e612d82ec1e4a1a) | report input changes with Interrupt Notify | 🟡 S4 | Section 5.4.1 (GPIO Channel Request, Table 18 - command 4 Interrupt Notify is initiated by the Producer, Set Values by the Consumer); Section 5.4, Table 16 (GPIO Channel Roles - Producer is Requester for GPIO Interrupts only) |
| [`b913cba37f`](https://github.com/maxdog988/OpenBIC/commit/b913cba37f7769dba393e5aec4aa9681b4e1d284) | encode Low as 0 and High as 1 | 🔴 H4 | Section 5.4.1 (GPIO Channel Request, Table 18 - Set Values GPIO Data: 0 = Low, 1 = High); Section 5.4.2 (GPIO Channel Response, Table 19 - GPIO Value: 0 = Low, 1 = High) |
| [`b3e3b376cd`](https://github.com/maxdog988/OpenBIC/commit/b3e3b376cd01d708c429fc9e0a90c410e323528e) | reject malformed requests from the host | 🟢 independent | Section 5.4.1 (GPIO Channel Request, Table 18 - Command byte followed by 2-byte GPIO Index/Data entries; Set Values data 2-15 Reserved); Section 5.2, Table 4 (0x01 Invalid command or parameter); Section 3.2, Table 1 requirement 12 (request-response model) |
| [`20ec513ef6`](https://github.com/maxdog988/OpenBIC/commit/20ec513ef691009a366f1c649679b847fb3fe47f) | report the driven SGPIO level for output pins | 🟢 independent | Section 5.4.2 (GPIO Channel Response, Table 19 - Get Values returns the GPIO Value); Section 5.15.2, Table 77 (GPIO_DIRECTION and GPIO_DEFAULT_OUTPUT of the GPIO Configuration Entry); Section 5.4, Table 17 (GPIO Channel-specific Status Codes) |
| [`14c4637a37`](https://github.com/maxdog988/OpenBIC/commit/14c4637a376bc4359d5b139dddcb26aea3f4dd0b) | reject Get requests whose response does not fit | 🟢 independent | Sections 5.4.1 and 5.4.2 (GPIO Channel Request/Response, Tables 18 and 19 - one entry per GPIO Index); Section 5.15, Table 65 (MAX_RESPONSE_PAYLOAD_SIZE); Section 5.2, Table 4 (0x06 Request size not supported) |
| [`45dbd60e2a`](https://github.com/maxdog988/OpenBIC/commit/45dbd60e2afd0e316c9eebc4f909e26b8174daf9) | reject unknown indexes and writes to input pins | 🟡 S1 | Section 5.4, Table 17 (GPIO Channel-specific Status Codes - 0x40 GPIO Index not supported, 0x42 Invalid operation for GPIO Configuration); Section 5.4.1 (GPIO Channel Request, Table 18 - GPIO Index 0-4095); Section 5.15.2, Table 77 (GPIO Configuration Entry - GPIO_INDEX, GPIO_DIRECTION) |
| [`ffe55799c4`](https://github.com/maxdog988/OpenBIC/commit/ffe55799c422b53e677a8defcbb69cbef3e4366e) | stop clearing an outstanding request by force | 🟢 independent | Section 3.2, Table 1 requirement 15 (only one outstanding request per communication channel); Section 5.4.1 (GPIO Channel Request, Table 18); Section 6.6 (Transport Guarantees - timeouts are implemented above OBMF-ICP) |
| [`9e853a987e`](https://github.com/maxdog988/OpenBIC/commit/9e853a987e1d9f275bb6599a84bde30bf2e73aad) | queue Interrupt Notify instead of dropping it while busy | 🟡 S4 | Section 5.4.1 (GPIO Channel Request, Table 18 - Interrupt Notify initiated by the Producer, one entry per GPIO); Section 5.15.1, Table 67 (GPIO_STATUS[1] INTERRUPT_PENDING - one or more GPIO interrupt notifications are pending); Section 3.2, Table 1 requirement 15 (one outstanding request per channel) |
| [`ce35e66673`](https://github.com/maxdog988/OpenBIC/commit/ce35e66673abfa08e8bdb0e6c964f146dd510278) | add a shell command to report input changes | 🟢 independent | Section 5.4.1 (GPIO Channel Request, Table 18 - Interrupt Notify initiated by the Producer); Section 5.15.1, Table 67 (GPIO_STATUS INTERRUPT_PENDING) |

**BMC (linux)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`be1c8757c9`](https://github.com/maxdog988/linux/commit/be1c8757c976f434becd487d44e0ce727e80fa22) | encode Low as 0 and High as 1 | 🔴 H4 | Section 5.4.1 (GPIO Channel Request, Table 18 - Set Values GPIO Data: 0 = Low, 1 = High); Section 5.4.2 (GPIO Channel Response, Table 19 - GPIO Value: 0 = Low, 1 = High) |
| [`b7ab97f609`](https://github.com/maxdog988/linux/commit/b7ab97f60904522474babd795ba683bf7114136d) | keep IRQ state when the device rejects it | 🔴 H5 | Section 5.4.1 (GPIO Channel Request, Table 18 - Set Interrupt Config, command 3); Section 5.4 (GPIO Channel-specific Status Codes, Table 17 - 0x41 Interrupt not supported for GPIO Index); Section 5.2 (any non-zero Status means the request failed) |
| [`c1991f6ba2`](https://github.com/maxdog988/linux/commit/c1991f6ba2f2d2906d3cbfbc9135591eaeb4b9e0) | free line names when chip registration fails | 🟢 independent | Section 5.15.2, Table 77 (GPIO Configuration Entry - GPIO_NAME, the source of the line names); Section 3.6 (Linux and OpenBMC Interoperability - GPIO channel exposed through the Linux GPIO model) |
| [`25222d68ec`](https://github.com/maxdog988/linux/commit/25222d68ecac676d947ab834badd5d16cd7e1c40) | map the GPIO channel status codes | 🟡 S1 | Section 5.4, Table 17 (GPIO Channel-specific Status Codes); Section 5.2, Table 4 (0x40-0x7F are defined per channel) |
| [`bc1c424afa`](https://github.com/maxdog988/linux/commit/bc1c424afa0399237e138642cd5c1abad5bc2d09) | address lines by their GPIO_INDEX | 🟡 S4 | Section 5.15.2, Table 77 (GPIO Configuration Entry - GPIO_INDEX: programmable index used by GPIO channel commands), Table 76 (GPIO_COUNT - 0 means no GPIOs exposed); Sections 5.4.1 and 5.4.2 (GPIO Channel Request/Response, Tables 18 and 19 - GPIO Index 0-4095); Section 5.4, Table 17 (0x40 GPIO Index not supported) |
| [`ca0df5452f`](https://github.com/maxdog988/linux/commit/ca0df5452fa4228aba81c8ff17e2d0670845dbf1) | report a failed Set Values request | 🟢 independent | Section 5.4 (GPIO Channel, Set Values request and response); Section 5.2, Table 4 (status carried in the response header) |

### Serial

**SMC1 (OpenBIC)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`cb430acf4c`](https://github.com/maxdog988/OpenBIC/commit/cb430acf4ce3f74cc4c3e4def62118c7e9510aa4) | keep each request within one OBMF message | 🟡 S2 | Section 5.5.1 (Serial Channel Request, Table 22 - Character Data up to the maximum allowed by the OBMF packet size, after a 3-byte header); Section 5.15, Table 65 (MAX_REQUEST_PAYLOAD_SIZE); Section 3.2, Table 1 requirement 26 (Device-specific message size) |
| [`7722df1ecb`](https://github.com/maxdog988/OpenBIC/commit/7722df1ecb7bbd91b4c627a6c14738e26cae43b7) | reject malformed requests from the host | 🟢 independent | Section 5.5.1 (Serial Channel Request, Table 22 - 3-byte header, Character Count of Character Data); Section 5.5.2 (Serial Channel Response, Table 23 - Accepted Character Count); Section 5.2, Table 4 (0x01 Invalid command or parameter); Section 3.2, Table 1 requirement 12 (request-response model) |
| [`e959dc47d7`](https://github.com/maxdog988/OpenBIC/commit/e959dc47d7799338b1ef7a763a874192597ce573) | correct the Transmit overrun and Carrier down bits | 🟢 independent | Section 5.5.1 (Serial Channel Request, Table 22 - Operation/Event: [0] Generate Break, [1] Break detected, [2] Transmit overrun, [3] Carrier down, [7:4] Reserved) |

**BMC (linux)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`c2daf0fe1f`](https://github.com/maxdog988/linux/commit/c2daf0fe1f374f51f99ebc2e876f066146f232d4) | bound each write to one OBMF message | 🟡 S2 | Section 5.5.1 (Serial Channel Request, Table 22 - Character Data up to the maximum allowed by the OBMF packet size, after a 3-byte header); Section 5.15, Table 65 (MAX_REQUEST_PAYLOAD_SIZE) |
| [`2925000200`](https://github.com/maxdog988/linux/commit/2925000200726cc3953ea7dc221eccce2f54748c) | reset the port count on hangup | 🟢 independent | Section 3.6 (Linux and OpenBMC Interoperability - channels follow the Linux TTY/UART model); Section 3.2, Table 1 requirement 22 (align with existing OpenBMC and Linux device driver models); Section 5.5 (OBMF Serial Channel) |
| [`d5da795155`](https://github.com/maxdog988/linux/commit/d5da795155a869290066f11a025d5cdf67e3aa74) | wait instead of failing on a full NACK | 🟢 independent | Section 5.5.2 (Serial Channel Response, Table 23 - NACK with Accepted Character Count; the sender retries starting with the first byte the receiver did not accept) and Figure 13 (Example Serial Channel flow) |
| [`8be70a98b3`](https://github.com/maxdog988/linux/commit/8be70a98b36b56d9abc11a02ede0604c20f001da) | bound the wait on a full NACK | 🟢 independent | Section 5.5.2 (Serial Channel Responses - Accepted Character Count, a full NACK); Section 6.6 (Transport Guarantees - timeouts are implemented above OBMF-ICP) |
| [`d95653374c`](https://github.com/maxdog988/linux/commit/d95653374cb93918be03cb7794ab6c883407ecb9) | map the Serial channel status codes | 🟡 S1 | Section 5.5, Table 21 (Serial Channel-specific Status Codes); Section 5.2, Table 4 (0x40-0x7F are defined per channel) |

### I2C

**SMC1 (OpenBIC)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`52cc0e3b96`](https://github.com/maxdog988/OpenBIC/commit/52cc0e3b96239d682c89fedbdf129e795a5b62b7) | reject oversized transfers instead of truncating | 🟡 S3 | Section 5.6.1 (I2C Controller Channel Request, Table 26 - Read Length and Write Data); Section 5.6.2 (I2C Controller Channel Response, Table 28 - Read Data); Section 5.2, Table 4 (0x06 Request size not supported); Section 5.15, Table 65 (MAX_RESPONSE_PAYLOAD_SIZE) |
| [`36700c9c4c`](https://github.com/maxdog988/OpenBIC/commit/36700c9c4c356b2b8eac240d5a9c3030187708e1) | report unclassified transfer errors as 0x04, not reserved 0x07 | 🟡 S1 | Section 5.2 (OBMF-ICP Status Codes, Table 4 - 0x04 Channel permanent error, 0x07-0x3F Reserved); Section 5.6, Table 25 (I2C Controller Channel-specific Status Codes 0x40-0x43) |
| [`fc9300533f`](https://github.com/maxdog988/OpenBIC/commit/fc9300533fd8b221d9258fd1143651c8cf694975) | advertise I2C_TARGET_BUFFER_SIZE and drop stale responses | 🟢 independent | Section 5.15.2, Table 80 (I2C Target Channel Configuration Data - 0x01 I2C_TARGET_BUFFER_SIZE, maximum length of an I2C Write Transaction); Section 5.7 (OBMF I2C Target Channel - the BMC emulates byte-level acknowledge up to the configured buffer); Section 3.2, Table 1 requirement 15 (one outstanding request per channel) |

**BMC (linux)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`c354bef1e3`](https://github.com/maxdog988/linux/commit/c354bef1e36e718b9bb0abb1da446484e8ff0dd0) | bound transfer length to one OBMF message | 🟡 S3 | Section 5.6.1 (I2C Controller Channel Request, Table 26 - Write Data starts at byte 4 after the 4-byte I2C header); Section 5.15, Table 65 (MAX_REQUEST_PAYLOAD_SIZE / MAX_RESPONSE_PAYLOAD_SIZE - per-channel message size limit); Section 3.2, Table 1 requirement 26 (Device may limit the maximum supported message size) |
| [`6f121815aa`](https://github.com/maxdog988/linux/commit/6f121815aaa3b21636481c25171864621d7df9e8) | size reads by MAX_RESPONSE_PAYLOAD_SIZE | 🟡 S3 | Section 5.15, Table 65 (MAX_RESPONSE_PAYLOAD_SIZE) and the I2C Controller Channel Request/Response layout (Sections 5.6.1 and 5.6.2). The adapter read and write limits follow the payload sizes the channel advertises, minus the I2C header. |
| [`579c72674d`](https://github.com/maxdog988/linux/commit/4a819005a5e5b0c3525cb1c6f3552c5744ea3b89) | fail short reads instead of returning stale data | 🟡 S3 | Section 5.6.2 (I2C Controller Channel Response, Table 28 - Read Length and Read Data carry the data read from the target); Section 5.6.1, Table 26 (Read Length - size of Read Data requested) |
| [`12bfa439c6`](https://github.com/maxdog988/linux/commit/39cd5242b0fb2aa84e67cffd47836c4f247b173a) | map the I2C Controller status codes | 🟡 S1 | Section 5.6, Table 25 (I2C Controller Channel-specific Status Codes); Section 5.2, Table 4 (0x40-0x7F are defined per channel) |

### MMIO

**SMC1 (OpenBIC)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`9023894946`](https://github.com/maxdog988/OpenBIC/commit/9023894946c9135550d7461388306dcc9ca1910a) | reject writes outside the local regions | 🟢 independent | Sections 5.3.3 and 5.3.7 (Short/Long Write Request, Tables 10 and 14 - 32-bit and 64-bit Address); Section 5.3, Table 7 (MMIO Channel-specific Status Codes - 0x40 Address out-of-range) |
| [`09bc69e6ec`](https://github.com/maxdog988/OpenBIC/commit/09bc69e6ec5b806033f044814252cc174d918024) | use the OBMF-ICP 1.0.0 MMIO message layout | 🔴 H1 | Section 5.3 (OBMF MMIO Channel, Table 5 - 2-byte header: Transaction in byte 0, byte 1 Reserved, payload at byte 2); Sections 5.3.3 and 5.3.7 (Short/Long Write Request, Tables 10 and 14 - Data follows the Address, no length field); Sections 5.3.2, 5.3.4, 5.3.6 and 5.3.8 (Short/Long Read/Write Response, Tables 9, 11, 13, 15) |
| [`546b988f08`](https://github.com/maxdog988/OpenBIC/commit/546b988f08ee38cffb66ea455c3671c123c4943d) | stop clearing an outstanding request by force | 🟢 independent | Section 3.2, Table 1 requirement 15 (only one outstanding request per communication channel); Section 5.3.7 (Long Write Request, Table 14); Section 6.6 (Transport Guarantees - timeouts are implemented above OBMF-ICP) |
| [`eeff6689f1`](https://github.com/maxdog988/OpenBIC/commit/eeff6689f182a3e1aca0eca21f9d4d7773770343) | report the MMIO window in the Configuration Data | 🟢 independent | Section 5.15.2, Table 75 (MMIO Channel Configuration Data - MAX_MMIO_READ_SIZE, MAX_MMIO_WRITE_SIZE, MMIO_WINDOW_BASE_ADDRESS, MMIO_WINDOW_SIZE); Section 5.15, Table 65 (CONFIG_SIZE) and Table 63 (Channel 0 Tables Convention - "r" read-only) |

**BMC (linux)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`d20d5816f4`](https://github.com/maxdog988/linux/commit/d6fc72df475d1d3cb091033e7303a6e2fc0f18e9) | fail short read responses | 🟢 independent | Sections 5.3.2 and 5.3.6 (Short/Long Read Response, Tables 9 and 13 - on Success the Data field carries exactly Read Length bytes) |
| [`929cc5d820`](https://github.com/maxdog988/linux/commit/b2a66f3099fc7784bc6106ddd45c8998042e5b83) | answer malformed POST code requests | 🟢 independent | Section 3.2, Table 1 requirement 12 (request-response model - every request gets a response); Section 5.3, Table 6 (MMIO Channel Roles - both roles act as Responder); Section 5.2, Table 4 (0x01 Invalid command or parameter for this channel) |
| [`8026aa619b`](https://github.com/maxdog988/linux/commit/d9d578d45c1248ecbf60787a56f77208e9a154d9) | parse device writes without a length field | 🔴 H1 | Section 5.3 (OBMF MMIO Channel, Table 5 - 2-byte header, byte 1 Reserved and set to 0); Sections 5.3.3 and 5.3.7 (Short/Long Write Request, Tables 10 and 14 - Data follows the Address, no length field); Sections 5.3.4 and 5.3.8 (Short/Long Write Response, Tables 11 and 15) |

### I3C

**SMC1 (OpenBIC)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`be927cf22f`](https://github.com/maxdog988/OpenBIC/commit/be927cf22fe7605f9bce5e2c3408307d222ff846) | answer DO_DAA when address assignment fails | 🟢 independent | Section 5.8.2 (I3C Controller Channel Response, Table 40 - DO_DAA response); Section 5.8, Table 34 (I3C Channel-specific Status Codes); Section 5.2, Table 4 (0x04 Channel permanent error); Section 3.2, Table 1 requirement 12 (request-response model) |
| [`0c61237488`](https://github.com/maxdog988/OpenBIC/commit/0c6123748831b6f65cdddbd2588536798042fe3b) | lay out the IBI target array as in the Configuration Data | 🔴 H9 | Section 5.15.2, Table 81 (I3C Controller Channel Configuration Data - 0x0E IBI_TARGET_COUNT, 0x0F IBI_TARGET_CONFIG_ARRAY) and Table 82 (IBI_TARGET_CONFIG_ARRAY - DYNAMIC_ADDR, CONTROL_FLAGS, CONFIGURED_MAX_IBI_PAYLOAD_LEN including the MDB, CONFIGURED_MAX_PENDING_READ_LEN; initial values set by the producer); Section 5.15, Table 65 (CONFIG_SIZE) |
| [`9f1c8dbc24`](https://github.com/maxdog988/OpenBIC/commit/9f1c8dbc240d5125c8da62fd611b613306918b26) | clamp host writes of the IBI target array | 🟢 independent | Section 5.15.2, Table 81 (I3C Controller Channel Configuration Data) and Table 82 (IBI_TARGET_CONFIG_ARRAY - AUTO_PENDING_READ_ENABLE, configured lengths; the producer must clamp values above its capability) |

**BMC (linux)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`986819961d`](https://github.com/maxdog988/linux/commit/7e5e64c5ad385b8967a57e3f5ac1cdfb1be47b05) | map the I3C Controller status codes | 🟡 S1 | Section 5.8, Table 34 (I3C Channel-specific Status Codes); Section 5.2, Table 4 (0x40-0x7F are defined per channel) |
| [`9ef6daff54`](https://github.com/maxdog988/linux/commit/d95e32cf2bed760ce1a6c3b92aaeeb9e39fe5bf2) | read the IBI target array of the Configuration Data | 🔴 H9 | Section 5.15.2, Table 81 (I3C Controller Channel Configuration Data - 0x0E IBI_TARGET_COUNT, 0x0F IBI_TARGET_CONFIG_ARRAY) and Table 82 (IBI_TARGET_CONFIG_ARRAY - DYNAMIC_ADDR, CONTROL_FLAGS, CONFIGURED_MAX_IBI_PAYLOAD_LEN including the MDB, CONFIGURED_MAX_PENDING_READ_LEN); Section 5.15, Table 65 (CONFIG_SIZE); Section 3.2, Table 1 requirement 29 (Channel 0 accessed with MMIO Short Read/Write) |

### SPI

**SMC1 (OpenBIC)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`929f51f9f6`](https://github.com/maxdog988/OpenBIC/commit/929f51f9f61a8ad39ea1100128d10d7234b31919) | reject reads larger than the response buffer | 🟡 S3 | Section 5.10.1 (SPI Controller Channel Request, Table 50 - Read Data Size); Section 5.15, Table 65 (MAX_RESPONSE_PAYLOAD_SIZE); Section 5.2, Table 4 (0x06 Request size not supported) |

**BMC (linux)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`8103c5cba4`](https://github.com/maxdog988/linux/commit/530d4c12f80fac36ac1dde54ab107a70150f3114) | honour the channel max payload sizes | 🟡 S3 | Section 5.15, Table 65 (MAX_REQUEST_PAYLOAD_SIZE / MAX_RESPONSE_PAYLOAD_SIZE - maximum Request/Response size for this channel); Section 3.2, Table 1 requirement 26 (Device-specific message size); Sections 5.10.1 and 5.10.2 (SPI Controller Channel Request and Response, Tables 50 and 51) |
| [`20cd128672`](https://github.com/maxdog988/linux/commit/5d781661845c1c737a7f24c7f292112e319fb758) | map the SPI Channel-specific status codes | 🟡 S1 | Section 5.10, Table 49 (SPI Channel-specific Status Codes); Section 5.2, Table 4 (generic status codes) |

### I/O

**SMC1 (OpenBIC)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`3fe18c068e`](https://github.com/maxdog988/OpenBIC/commit/3fe18c068e93650bf82c83ebe8a1164d246b4fa4) | drop the stray status byte from read responses | 🔴 H7 | Section 5.11.3 (Sequential Port I/O Read 8-bit Response, Table 56 - the payload is the Data only); Section 5.1, Table 3 (Status is carried in byte 1[6:0] of the OBMF-ICP message header) |
| [`f9baa1a6f1`](https://github.com/maxdog988/OpenBIC/commit/f9baa1a6f1fd05984382cd2beaf485695410bb3d) | drop the Tag from the I/O channel header | 🔴 H6 | Section 5.11 (OBMF I/O Channel, Table 52 - 1-byte header: Transaction[3:0], Reserved[7:4], payload at byte 1, no Tag); Sections 5.11.2-5.11.5 (Tables 55-58); Section 3.2, Table 1 requirement 15 (one outstanding request per channel) |
| [`93e9107248`](https://github.com/maxdog988/OpenBIC/commit/93e91072481a0d17663847ad34d376a925320e4c) | bound read responses and reject writes to unserved ports | 🟡 S3 | Sections 5.11.2-5.11.5 (Sequential Port I/O Read/Write, Tables 55-58) and 5.11.10-5.11.15 (Fixed Port I/O); Section 5.11, Table 54 (IO Channel-specific Status Codes - 0x40 Address out-of-range); Section 5.2, Table 4 (0x06 Request size not supported); Section 5.15, Table 65 (MAX_RESPONSE_PAYLOAD_SIZE) |
| [`28d4706b6f`](https://github.com/maxdog988/OpenBIC/commit/28d4706b6f21e8ef40433e82866ef6b5d58bb52d) | stop clearing an outstanding request by force | 🟢 independent | Section 3.2, Table 1 requirement 15 (only one outstanding request per communication channel); Section 5.11.4 (Sequential Port I/O Write Request, Table 57); Section 6.6 (Transport Guarantees - timeouts are implemented above OBMF-ICP) |

**BMC (linux)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`9e86f49106`](https://github.com/maxdog988/linux/commit/3da413fd7cbbbbfbfd98f5ae72c6b038dccebebd) | range-check Fixed Port I/O on a single port | 🟢 independent | Sections 5.11.10-5.11.15 (Fixed Port I/O transactions - all accesses target the same address); Section 5.15.2, Table 86 (IO_RANGE_CFG - IO_RANGE_START/IO_RANGE_END) |
| [`5c592a59c6`](https://github.com/maxdog988/linux/commit/7aa0469af07a48188ff539edb86ddbeb6a9daa26) | fail short read responses | 🔴 H7 | Section 5.11.3 (Sequential Port I/O Read 8-bit Response, Table 56 - on Success the Data field carries Size bytes), applied to the 16/32-bit and Fixed Port variants by Sections 5.11.6-5.11.14 |
| [`48acaa9d26`](https://github.com/maxdog988/linux/commit/8a2373514f37aab58b935e039343a9fe32f7413d) | answer malformed POST code requests | 🟢 independent | Section 3.2, Table 1 requirement 12 (request-response model - every request gets a response); Section 5.11.4 (Sequential Port I/O Write 8-bit Request, Table 57 - Size and Data); Section 5.2, Table 4 (0x01 Invalid command or parameter for this channel) |
| [`b85a875d6e`](https://github.com/maxdog988/linux/commit/90e474d11f963b75aa6b5a61cada4577ca70cb6d) | drop the Tag from the I/O channel header | 🔴 H6 | Section 5.11 (OBMF I/O Channel, Table 52 - 1-byte header: Transaction[3:0], Reserved[7:4], payload at byte 1, no Tag); Sections 5.11.2-5.11.5 (Tables 55-58); Section 3.2, Table 1 requirement 15 (one outstanding request per channel); Section 3.3 (transport guarantees in-order delivery) |

### Memory Device

**SMC1 (OpenBIC)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`25264f1b8a`](https://github.com/maxdog988/OpenBIC/commit/25264f1b8a6f3b1c786c62f6fbbcc86653c98cba) | stop using the reserved status code 0x07 | 🟡 S1 | Section 5.2 (OBMF-ICP Status Codes, Table 4 - generic codes end at 0x06, 0x07-0x3F Reserved); Section 5.12, Table 60 (Memory Device Channel-specific Status Codes) |
| [`461da2dc0c`](https://github.com/maxdog988/OpenBIC/commit/461da2dc0c3fedd9510781cb8e9a23090cbce2ba) | take read data from the start of the response payload | 🔴 H8 | Section 5.12.2 (Memory Device Channel Response, Table 62 - Data on a successful Read, omitted otherwise); Section 5.1, Table 3 (Status in byte 1[6:0] of the message header) |
| [`aa6d3a78ab`](https://github.com/maxdog988/OpenBIC/commit/aa6d3a78abfd6cdc7a184f0b15ef026c35c4551f) | drop a stale response before sending a request | 🟡 S5 | Section 3.2, Table 1 requirement 15 (one outstanding request per channel) and requirement 12 (request-response model); Section 6.6 (Transport Guarantees - timeouts are implemented by the layers above OBMF-ICP); Section 5.12 (OBMF Memory Device Channel) |
| [`b8e6b15c4f`](https://github.com/maxdog988/OpenBIC/commit/b8e6b15c4f2beff53b6d170e2bfc74ac41a713c4) | add shell commands to exercise the channel | 🟢 independent | Section 5.12.1 (Memory Device Channel Request, Table 61 - Memory Read, Write and Erase, issued by the Consumer); Section 5.12, Table 59 (Memory Device Channel Roles - the Consumer is the Requester) |

**BMC (linux)**

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`9b52d5c7a8`](https://github.com/maxdog988/linux/commit/4022017e68c0f43e8fcc7cb7fd054dd691c57a78) | map the Memory Device status codes | 🟡 S1 | Section 5.12, Table 60 (Memory Device Channel-specific Status Codes); Section 5.2, Table 4 (0x40-0x7F are defined per channel) |
| [`43c06cd020`](https://github.com/maxdog988/linux/commit/79c4d38e1f5652e70a2beea069ebd4ec35bbaab2) | carry only the read data in responses | 🔴 H8 | Section 5.12.2 (Memory Device Channel Response, Table 62 - Data only when Status is Success and Command is Read, otherwise omitted); Section 5.1, Table 3 (Status in byte 1[6:0] of the message header) |
| [`be90297e15`](https://github.com/maxdog988/linux/commit/14276b81ccc8f3994f91f928ae498d89a740ddcc) | enforce MEM_DEV_CFG read/write size limits | 🟡 S3 | Section 5.12 (Memory Device Channel, MEM_DEV_CFG and status codes) |

### USB class and platform (zephyr_nuvoton, SMC1 only)

Repository [maxdog988/zephyr](https://github.com/maxdog988/zephyr), branch `2.6-dev`. Commits are in apply order. Spec names are from OBMF-ICP over USB 1.0 unless stated.

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| [`0f9a650352`](https://github.com/maxdog988/zephyr/commit/0f9a6503526abc692b5917c89cdf12d53b3f8d6e) | usb: Support OBMF over USB 1.0rc5 | 🟢 baseline, used by OpenBIC | Section 5.4 (USB Descriptors for OCP OBMF-ICP Interface: class 0xEF, subclass 0x09, protocol 0x01, functional descriptor, Bulk IN/OUT) |
| [`28b727f85e`](https://github.com/maxdog988/zephyr/commit/28b727f85e364a341ac2bce7693720bf18dec02b) | obmf: Support OBMF-ICP v0.8 WIP | 🟢 baseline | Section 5.4 (descriptor layout) |
| [`deb98128a8`](https://github.com/maxdog988/zephyr/commit/deb98128a871e8405340071fc4837673cc36ce89) | usb: obmf: add bus_reset callback to reset channel state on host disconnect | 🟢 baseline, used by OpenBIC `.bus_reset` | Error recovery (host-side USB PORT RESET / BUS RESET; the device resets its state) |
| [`e69bb6a9a0`](https://github.com/maxdog988/zephyr/commit/e69bb6a9a06c10415ead9e1ff3c9fe6e687d127d) | soc: npcm400f: Add CR_UART3 support | 🟢 baseline, needed by the Serial channel to SMC2 | none (SoC support) |
| [`121b43169d`](https://github.com/maxdog988/zephyr/commit/121b43169d98aeba3bacff66d003850a3cc7137b) | obmf: Add link down/up handling for USB suspend/resume | 🟢 baseline, used by OpenBIC `.link_down` / `.link_up` | none (link-state handling; the retrievable spec text does not define it) |
| [`a76c665b3d`](https://github.com/maxdog988/zephyr/commit/a76c665b3de2203e7838bcd4a50750b881357f3e) | drivers: i3c: npcm4: add a Hot-Join completion callback | 🟢 baseline, used by `obmf_i3c.c` | none (I3C driver support for the I3C channel) |
| [`74b7a48968`](https://github.com/maxdog988/zephyr/commit/74b7a489683684bd0b71de33dc733daa998c9811) | drivers: i3c: npcm4: keep pending TX data across a controller write | 🟢 independent | none (I3C driver: an IBI response must survive a private write between the IBI and the read) |
| [`0efcba6eea`](https://github.com/maxdog988/zephyr/commit/0efcba6eea0eb71c2f1cc762b65efdb6e194f216) | usb: class: obmf: flush the Bulk IN endpoint when a write fails | 🟢 independent | Section 5.15 (Channel 0 discovery must complete again after the host returns) |
| `67560c4650` | usb: class: obmf: report OBMF-ICP 1.0 in bcdOCPOBMFVersion | 🟢 independent (the BMC stores the field and does not check it) | Section 5.4 (bcdOCPOBMFVersion is the OBMF-ICP release, not the USB document version) |
| `f1760d5dfb` | usb: class: obmf: reset OBMF state on CLEAR_FEATURE(ENDPOINT_HALT) | 🟢 independent | Error recovery (on clearing a STALL the device must flush its pipelines and reset its counters) |
| `b091b68916` | usb: class: obmf: NAK Bulk OUT instead of dropping when the app is full | 🟢 SMC1-internal, pairs with OpenBIC `2b1cbf93de` (Z1) | Bulk OUT endpoint (a message may span several packets up to wMaxWrTransferSize; the device may NAK until it can accept the next packet) |

Notes: the first six zephyr commits and `74b7a48968` are prerequisites of the OpenBIC range: the application already registers `.bus_reset`, `.link_down` and the Hot-Join callback. Every commit above except `4f1f8b5d2a` (SoC download mode, `0001-soc-...`) is also stored as a patch in `patches/` (`0001` to `0011`). The OpenBIC build applies a patch only when no zephyr commit with the same subject exists, so a fresh zephyr tree gets all of them. Applying the 12 patches in order on `3be104f3b5` gives exactly the current `2.6-dev` tree.

**SMC1 (OpenBIC), transport and platform**

These two OpenBIC commits are not tied to one channel, so they sit at the end of the OpenBIC history.

| Commit | Summary | Pairing | Spec |
|---|---|---|---|
| `2b1cbf93de` | obmf: apply USB backpressure when the RX ring buffer is full | 🟢 SMC1-internal, needs zephyr `b091b68916` (Z1) | Section 5.1 (message transport over USB); the host is NAKed instead of bytes being dropped |
| `40fa8f3e00` | smc-npcm400-obmf: report a per-device MCTP endpoint UUID | 🟢 independent (Z2) | DMTF DSP0236, Get Endpoint UUID (not an OBMF section) |

## 5. Ordering inside one repository

Some commits use a constant or helper added earlier, so they must keep their relative order (for example when cherry-picking).

**SMC1**

- Request timeout before reassembly hardening: `f872` then `72d9`; `0994` then `72d9`.
- CHANNEL_CONTROL before access-type checks: `38e0` then `146a`.
- GPIO: `b913` (High=1) then `b3e3` (data validation); `9f8e` then `50ea` then `9e85` then `ce35`; `45db` before `9e85`.
- Removing the forced clear of an outstanding request (`gpio`, `mmio`, `io`) needs the Channel 0 timeout commit `f872` first.
- I/O and Memory Device: `3fe1` then `f9ba`; `f9ba` and `3670` before `2526` (it deletes `OBMF_ICP_ERR_OTHER`); `2526` then `461d`.
- I3C: `0c61` then `9f1c`.

**BMC**

- `799b` and `471a` before `20d2`.
- `be4c` then `a880` then `8fa2`.
- `c199` and `b7ab` before `bc1c`; `2522` before `d956`.
- `929c` before `8026`; `48ac` before `b85a`; `4996` before `579c`.

## 6. Known open points

| # | Where | Description |
|---|---|---|
| 1 | SMC1 and BMC | Fixed in `6f121815aa`: the BMC sizes I2C reads from the advertised MAX_RESPONSE_PAYLOAD_SIZE (248 data bytes for SMC1), so larger reads are refused locally. |
| 2 | SMC1 GPIO | `OBMF_GPIO_MAX_INT_INDEX` is 128, while the spec index range and the BMC allow more. Higher indexes cannot be armed. |
| 3 | BMC Channel 0 | Fixed in `908c48bfe3`: a fragmented request to an unknown channel is answered once, by following the Size field. |
| 4 | BMC GPIO and Serial | Fixed in `ca0df5452f` (a failed Set Values is logged and the cached output value restored; `gpio_chip.set` cannot return an error in 6.12) and `8be70a98b3` (the full-NACK retry gives up after 5 s with `-ETIMEDOUT`). |
| 5 | SMC1 Channel 0 | Fixed in `6d812f052e`: RESET on CHANNEL_CONTROL clears the reassembly discard state. |
| 6 | SMC1 Channel 0 | Fixed in `6d812f052e`: a bad-channel request is answered once even when its last fragment is exactly one packet long. |
| 7 | BMC Channel 0 | Fixed in `b1edeb28ce`: the first OBMF_VER read is retried, which removes the intermittent `discovery failed: -110` on unbind and bind. |
| 8 | SMC1 USB | Fixed in zephyr `b091b68916` and OpenBIC `2b1cbf93de`: a full RX ring buffer used to drop bytes mid-message and desynchronise the framing. The packet now stays in the endpoint FIFO and the host is NAKed. Tested with a simulated full ring (about 6750 NAK events, 0 failures); real traffic never filled even a 520-byte ring, so the natural case is not exercised. |
| 9 | SMC1 USB | Fixed in zephyr `67560c4650`: `bcdOCPOBMFVersion` was 0x0080 and is now 0x0100 (OBMF-ICP 1.0). The BCD layout is inferred; the retrievable spec text does not give it. |
| 10 | SMC1 and SMC2 MCTP | Fixed in OpenBIC `40fa8f3e00`: every SMC returned the same UUID, so mctpd merged SMC2 into SMC1. The UUID now includes `OBMF_MCTP_UUID_NODE`. **SMC2 still runs the prebuilt image with the old UUID.** Rebuild it with a different node id and update it by PLDM. |
| 11 | BMC meta layer | `mctp-config` sets MTU 243 on `mctpi2c27` and `mctpi3c2` each time mctpd restarts (`ExecStartPost`), which overrides the 240 set by the udev rule. Use 240 there too. Not fixed. |
| 12 | BMC MCTP | A stale route `eid 8 ... dev mctpusb0` remains next to the BMC's own EID 8. Not investigated. |
| 13 | USB spec | Not retrievable from the knowledge base: zero-length-packet and short-packet termination rules, and the numeric descriptor type and subtype values (SMC1 uses 0x24 and 0x01). |

## 7. Hardware test summary

Tested on an NPCM845 EVB (BMC) with an NPCM400 EVB (SMC1) and SMC2 behind SMC1.

- I2C: 8 TMP75 sensors, TCA6416, NCT7363 and PCA9848 behind BMC i2c-30. 660 reads, 0 failures.
- I3C, SPI, Memory Device (256-byte read/write), GPIO and the Serial channel (SMC2 console on port 2231) all pass.
- Driver unbind and rebind with a request in flight recovers after the Bulk IN flush fix.
- Not covered by hardware tests: crafted fragment sequences (stale, header mismatch, overflow), the stalled-reply timeout path, and large SPI or I3C transfers.

### 7.1 Stress test (`obmf-stress`)

`obmf-stress` is a shell script installed on the BMC as `/usr/bin/obmf-stress` (openbmc meta layer, `meta-evb-npcm845`, recipe `obmf-stress`). It uses only the kernel interfaces of the driver, so it needs no SMC1 shell.

```
obmf-stress -b 27 -a 0x10 -n 0x66 -d 60 -r 10 -t
```

| Option | Meaning |
|---|---|
| `-b BUS` / `-a ADDR` | OBMF I2C adapter (i2c-27) and a target that answers (SMC2, 0x10) |
| `-n ADDR` | Unused address (0x66) that must NACK; the NACK must reach Linux as ENXIO |
| `-d SECS` | Length of the parallel phase |
| `-r COUNT` | Number of driver unbind and bind cycles |
| `-t` | Also write to the Serial channel (`/dev/ttyOBMF0_0`) |

Phases:

1. **Discovery check.** The script finds the OBMF interface and the I2C, GPIO, SPI-MTD and tty devices that the 19 channels created.
2. **Functional checks per channel.** I2C: 4-byte and 248-byte reads, NACK at the unused address, and a 253-byte write that must be refused (one OBMF message). GPIO: `gpioget` on every chip. SPI: two 8 KiB MTD reads must be identical. Serial: a 64 KiB write to the tty.
3. **Parallel stress.** For `-d` seconds one loop per channel type runs at the same time: I2C reads on i2c-27, GPIO input reads, SPI-NOR MTD reads (data compared with the first read) and tty writes. Every loop counts iterations and failures, and any failure fails the run. The script finishes by checking the kernel log for oops and warnings.
4. **Rebind loop.** `echo $IFACE > /sys/bus/usb/drivers/obmf/unbind` followed by `bind`, `-r` times. Each bind repeats the whole Channel 0 discovery and must succeed. After the loop the script runs discovery again and repeats the I2C read and the SPI data check.

Result with the current history (60 s, 10 rebinds): all checks pass, with about 4000 I2C, 3000 GPIO, 1000 SPI and 600 tty iterations and no failures. Before `b1edeb28ce`, 1 in 5 rebinds failed with `discovery failed: -110`; after it, 10 of 10 pass.

NACK address: use an address `i2ctransfer` accepts (0x03-0x77), for example 0x66. 0x7f is rejected by the tool itself, which looks like a driver failure but is not.

### 7.2 PLDM firmware update of SMC2

The package `SMCNPCM_signed_with_header.bin` is fetched from the TFTP server (`tftp -g -r SMCNPCM_signed_with_header.bin -l /tmp/pkg.bin 192.168.11.1`) and sent with the pldmd `StartUpdate` D-Bus method:

**Choosing the target device.** The device to update is selected by two things: the MCTP interface (the bus the device sits on) and the physical address passed to `SetupEndpoint`. Fill in the variables below for your target, then run the same commands.

| Target | `IFACE` (MCTP interface) | `SetupEndpoint` address argument | Where the value comes from |
|---|---|---|---|
| SMC2 on I2C | `mctpi2c27` | `ay 1 0x10` (1 byte: 7-bit I2C address) | SMC2 I2C address; `mctpi2c<N>` is the BMC I2C bus number of the OBMF I2C channel (`ls /sys/class/net \| grep mctp`) |
| SMC2 on I3C | `mctpi3c2` | `ay 6 0x06 0x32 0x12 0x34 0x56 0x07` (6 bytes: I3C PID) | SMC2 I3C PID; for this board it is in `npcm400_evb_i3c_obmf.json` |

```
IFACE=mctpi2c27                      # target interface, from the table above
ADDR="ay 1 0x10"                     # target address, from the table above

mctp link set $IFACE up              # MTU 240 is set by the udev rule, see below
busctl call au.com.codeconstruct.MCTP1 /au/com/codeconstruct/mctp1/interfaces/$IFACE \
    au.com.codeconstruct.MCTP.BusOwner1 SetupEndpoint $ADDR
# the reply is "yisb <eid> <net> <path> <new>": note the EID of the device
mctp endpoint                        # check: the EID is listed on the expected interface
systemctl restart pldmd              # pldmd then creates the software object for the endpoint
busctl tree xyz.openbmc_project.PLDM # lists one software object per PLDM-capable endpoint
busctl call xyz.openbmc_project.PLDM /xyz/openbmc_project/software/pldm \
    xyz.openbmc_project.Software.Update StartUpdate hs 3 \
    xyz.openbmc_project.Software.ApplyTime.RequestedApplyTimes.Immediate 3 < /tmp/pkg.bin
```

For I3C use `IFACE=mctpi3c2` and `ADDR="ay 6 0x06 0x32 0x12 0x34 0x56 0x07"`.

How the target is picked, and what to check:

- `SetupEndpoint` only assigns an EID to the device at that address on that interface. `StartUpdate` itself takes no EID: pldmd matches the package against every endpoint it knows. To update one device only, make sure no other PLDM-capable endpoint is set up (`mctp endpoint`; remove extras with `busctl call au.com.codeconstruct.MCTP1 /au/com/codeconstruct/mctp1/endpoints/<eid> au.com.codeconstruct.MCTP.Endpoint1 Remove`), or use a package whose descriptors match only that device.
- Each device must report a unique endpoint UUID (build with `-DOBMF_MCTP_UUID_NODE=<n>`, different for every SMC). mctpd identifies endpoints by UUID: if two devices share a UUID, `SetupEndpoint` on the second interface returns the first device's EID and the route stays on the first interface, so no traffic goes to the device you selected.
- After `SetupEndpoint`, confirm in `mctp route` that the EID is routed through the interface you chose.

| Link | MTU | Result | Time |
|---|---|---|---|
| i2c-27 (MCTP over I2C, SMC2 at 0x10) | 240 | transfer, verify and apply complete | about 93 s |
| i3c-2 (MCTP over I3C) | 240 | transfer, verify and apply complete | about 8 s |

Notes:

- The default MCTP-over-I2C MTU of 254 gives 258-byte I2C writes. The BMC driver refuses them, because SMC1 advertises a 252-byte maximum request payload (Section 5.15, Table 65, MAX_REQUEST_PAYLOAD_SIZE). Set the interface MTU to 240 or less. Commit `ad645b34eb` in the openbmc meta layer does this: a udev rule (`91-mctp-obmf-mtu.rules`) sets MTU 240 on `mctpi2c*` and `mctpi3c2` when the OBMF driver creates them, because `mctp-config` runs before they exist.
- The package also matches the SMC1 USB endpoint (EID 11). An update that includes it replaces the SMC1 OBMF image with a plain MCTP-over-USB image. Take `mctpusb0` down (`mctp link set mctpusb0 down`) before the update, or reflash SMC1 with `west flash` afterwards.
- Short `__i2c_transfer failed -6` and `-16` messages appear in dmesg while SMC2 writes flash. They are retried by MCTP and do not fail the update.

### 7.3 i2c-30 sensors

`nuvoton_npcm400_evb_i2c-obmf.json` (installed as `npcm400_evb_i2c.json` on the obmf distro) exposes 8 TMP75 sensors (`SMC2_I2C30_Temp1` to `Temp8`, buses 33-40) and the NCT7363 fan (`SMC2_NCT7363_Fan1`, type I2CFan, bus 30, address 0x23) so that dbus-sensors keeps reading them. Commit: `f63193f1c8` in the openbmc meta layer. All 9 sensors read on a cold boot with the fixed fan entry (no Connector, because the NCT7363 has no `pwm1_enable` and `fansensor` aborts if it is asked to write one).

### 7.4 USB class driver review (BMC and SMC1)

Both drivers were checked against the `obmf-icp-over-usb-1.0` document in the knowledge base.

| Item | BMC (`drivers/usb/class/obmf/`) | SMC1 (`zephyr_nuvoton/subsys/usb/class/obmf.c`) | Result |
|---|---|---|---|
| Class, subclass, protocol | matches 0xEF, 0x09 | 0xEF, 0x09, 0x01 | conformant |
| Bulk IN and OUT | required at probe | present | conformant |
| Interrupt endpoints | optional, used only if present | disabled, sizes advertised as 0 | conformant |
| bMultimessageSupport | rejects anything but 0x00 | 0x00 | conformant |
| wMaxWr/RdTransferSize | parsed, sets the transfer sizes (512/512) | 512 at high speed, 64 otherwise | conformant |
| bcdOCPOBMFVersion | stored, not checked | 0x0100 | conformant after `67560c4650` |
| Halt and reset recovery | `usb_clear_halt`, `pre_reset` and `post_reset` | RX buffer and application state cleared on halt clear, reset and disconnect | conformant after `f1760d5dfb` |
| RX overflow | not applicable | NAK instead of drop | fixed by `b091b68916` |

### 7.5 MCTP over I2C, I3C and USB (SMC1 and SMC2)

- mctpd identifies an endpoint by its UUID. With the old identical UUID, `SetupEndpoint` on `mctpi2c27` returned the existing EID reached over `mctpusb0`, nothing was sent over I2C, and `mctpi2c27` showed `TX dropped` with a single RX packet. After removing the endpoints and discovering again, SMC2 answered at 0x10.
- The I3C physical address for `SetupEndpoint` is the target PID (`06:32:12:34:56:07` for SMC2), not the BMC controller's PID.
- After OpenBIC `40fa8f3e00` and an mctpd restart: SMC1 is EID 11 over USB with UUID `4e50434d-3430-4000-8000-4f4200000001`, SMC2 is EID 9 over `mctpi2c27` and EID 10 over `mctpi3c2`. No traffic test was run over these routes afterwards.
- Check the endpoints with `mctp route` and `busctl tree au.com.codeconstruct.MCTP1`. SMC2 still reports the old UUID (open point 10).

### 7.6 Overnight stress run

The all-night run was lost: the host PC restarted at 09:12 and killed the runner. Only cycle 1 (38 checks, 0 failures) and a partial cycle 2 exist. The partial cycle 2 logged `Serial partial NACK ... retry 3/3` messages and 9 dropped messages during parallel stress. It is unanalysed. Run `obmf-night/run.sh` on a machine that does not restart to repeat it.

## 8. Commit rules used in this history

- Subject names the channel with no brackets, for example `obmf: ch0: ...` or `usb: class: obmf: gpio: ...`.
- Body explains the change and ends with `Reference: OCP OBMF Spec 1.0.0, Section X.Y` and `Signed-off-by`.
- Commits are grouped by channel and never interleave channels.
- zephyr_nuvoton commits use `usb: class: obmf: ...` or `drivers: i3c: npcm4: ...`, with the same `Signed-off-by`.
