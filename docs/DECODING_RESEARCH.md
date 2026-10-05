# R4W/R8W decoding research

**Scope:** R8W `v142.113.127` and R4W `v127.128.123`, with database
`260702`. The main/DSP/GPS payloads have not been decoded. This document records
experiments and the evidence needed to resolve them; it does not claim that a
failed hypothesis proves encryption or identifies a cipher.

## Reproduce the offline probes

```sh
python3 tools/rseries_probe.py /path/to/R8W_v142.113.127_db260702.bin
python3 tools/rseries_probe.py /path/to/R4W_v127.128.123_db260702.bin
```

This read-only command reports byte entropy, aligned 4/8/16/32-byte block
repetition, and candidate Cortex-M vectors under two address-map assumptions:
SRAM at `0x20000000`, code at zero or `0x08000000`. It tests all 256 legacy
transpose/subtract keys and 16 periodic XOR/subtract hypotheses per component.
The periodic hypotheses use the most frequent aligned 8/16-byte block as
possible uniform zero/FF padding, on raw and transposed data. A candidate vector
would still require valid disassembly, references and decoded structures before
being accepted. Other memory maps, a header preceding the vector table, nonuniform
padding or a different transform are outside these tests.

Measured results:

| Payload | R8W entropy (bits/byte) | R8W repeated 16-byte occurrences | R4W entropy | R4W repeated 16-byte occurrences |
|---|---:|---:|---:|---:|
| Main | 7.9956 | 738 | 7.9747 | 1814 |
| DSP | 7.9981 | 344 | 7.9966 | 323 |
| GPS | 7.9947 | 95 | 7.9945 | 88 |
| Alternate DSP (`N2DS`) | 7.9980 | 340 | 7.9968 | 325 |

Repeated occurrences count copies beyond the first occurrence of each block.
Every component produced **zero candidate vectors in these 272 probes**. The
probe tests include positive controls for real-shaped legacy-encoded vectors
and a periodic XOR fixture so a probe failure is not just a nonworking detector.

The repeated aligned 16-byte blocks and near-uniform byte distribution support
investigating a deterministic block transform. They do not prove AES-ECB:
other block transforms, encoding stages and separately reset cipher modes can
also repeat. Trying the R7 keys alone is insufficient.

## Updater investigation

The Windows Uniden R Series Tool v2.26 and the installed Mac updater were
inspected locally. No executable, vendor disassembly or extracted asset is
included in this repository. The following names identify methods in the
Windows updater, not API contracts implemented by this toolkit:

- `FWDloadFormat`: legacy sound/database keys and model-specific MCU ID strings.
  Its AES database flag identifies a format, not a recovered decryption key.
- `FWUpdate.DloadCheckMCUID`: matches the device's returned MCU ID to the
  component file offset and page count. R4W/R8W each have both `Nuv` and `Nuv2`
  DSP matches. The latter selects `dspNu2FileOffset`/`dspNu2FileLength`, consistent
  with `N2DS`; the actual hardware population must be checked on a unit.
- `FWUpdate.UpdateMCUSW`: its transfer loop copies a page directly from
  `FWFileInfo.fileData`, computes a transport checksum and sends it through
  `UARTCommUtils.WriteBytes`. This method does not decrypt the copied code page.
  A transport checksum is not evidence that arbitrary modified firmware is accepted.
- `ReadRDVersionInfo.UserSettingRead` and `VersionInfoRead`: provide concrete
  read-path targets for device investigation. Version reads involve beacon,
  sync, command, ready/ACK, data and end exchanges; blindly sending a command
  string without its surrounding handshake is insufficient.
- `UserSettingR4W/v127` and `UserSettingR8W/v142`: contain version-specific
  settings serialization, including K/Ka block fields. Field names alone do
  not establish that the detector menu omits the setting.

A local candidate-key scan tested every byte offset of the Windows executable,
Mac executable and R8W wireless payload as contiguous AES-128/192/256 key
material, and as key bytes stored one per 32-bit word. Four repeated main-code
blocks (two per model, including the repeated early-vector-region block) were
used as targets. None decrypted to a uniform 16-byte value. This scan assumes
ordinary key storage, direct AES input and uniform plaintext: it does **not**
exclude byte-swapped keys/data, derived keys, expanded schedules, runtime-loaded
keys, other ciphers or additional transforms. No claim of an exhaustive AES
key search is made.

A follow-up AES-128 scan also tested original, reversed and 32-bit-word-swapped
key layouts, with original/reversed/word-swapped/legacy-transposed ciphertext
blocks on the same three files. It likewise found no uniform-padding match.
A synthetic AES-encrypted-zero fixture with a deliberately embedded key was
used to verify the scanning code's positive path. These extra representation
checks still leave derived keys, expanded schedules and non-AES transforms open.

**Inference:** the observed direct transfer path makes the device-side update
handler/bootloader a strong next target for decoding. It does not prove where
every transformation occurs; the firmware files alone do not expose a verified
bootloader decoder.

## What an R8W USB connection can establish

Start with USB identity, the serial interface and the updater's established
read transactions. Record raw replies and the actual model/version before
interpreting settings; keep device captures local because they can contain
identifiers and saved locations.

1. Read component versions and any accessible hardware/MCU identification.
   The DSP identity could resolve whether this unit uses the primary or `N2DS`
   image, which cannot be inferred from the filename.
2. Read the settings blob with the stock protocol, retaining an unchanged raw
   baseline. Compare its decoded fields with the unit's own menus. That can
   establish whether an apparent extra parameter is really exposed or merely
   reserved, obsolete or unsupported.
3. Determine whether a **documented read path** exists for further data. No
   firmware/bootloader readback has been verified over this interface. Do not
   infer one from an update/download command, and do not fuzz unknown commands
   as part of a read-only session.

USB access by itself is not a bootloader dump or a decryption-key extractor.
If the update handler lives in a separate protected boot region, recovering it
may require a readable older image, a supported readback interface or separate
hardware-debug work. Opening the case, changing protection bits, entering erase
operations and flashing test images are separate actions, not prerequisites
for the initial USB identification/settings investigation.

The wireless interface is another read target, distinct from USB. An independent
[R8W BLE capture project](https://github.com/AegisX86/UnidenR8wlink/blob/main/PROTOCOL.md)
reports readable version/settings characteristics and alert telemetry on one
older R8W firmware. It does not establish a USB memory-read interface, a
controller decoder, or compatibility with the current R4W image.

## Acceptance criteria for a decoder

A successful decoder must produce credible code/vector tables across the
main, DSP and GPS images, establish how model/component keys are selected,
validate decoded references and structures, and re-encode byte-for-byte against
the originals. Hidden key combinations require tracing the button handler or
confirming them on a unit; printable text or updater labels alone are insufficient.
Until those checks pass, the toolkit retains opaque bytes and refuses W-model
controller edits.
