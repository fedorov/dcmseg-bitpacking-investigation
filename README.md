# Investigation: Single-Bit Encoding of Binary DICOM Segmentations

## Context

This investigation is triggered by [highdicom issue #393](https://github.com/ImagingDataCommons/highdicom/issues/393), where a user reports that binary segmentations with dimensions not divisible by 8 (specifically 187×239) show "frames increasingly shifted in the X axis as slice number increases" when viewed in 3D Slicer, MicroDicom, and Weasis. OHIF renders correctly.

The libraries analyzed are at the versions checked into this repository's submodules, which represent very recent (late 2024/2025) versions. Weasis's external dependencies (dcm4che3 and weasis-dicom-tools) are also included as submodules.

## DICOM Standard Requirements (Part 5, [Section 8.1.1](https://dicom.nema.org/medical/dicom/current/output/chtml/part05/chapter_8.html#sect_8.1.1))

The standard specifies for 1-bit-per-pixel data:

1. **Bit ordering**: LSB-first — the first pixel is the least significant bit of the first byte
2. **Continuous packing across frames**: In multi-frame images, bits are packed **continuously across frame boundaries**. A frame other than the first may start in the middle of a byte.
3. **Padding**: Padding to fill the last byte and ensure even Value Field length is applied **only at the very end of the complete PixelData**, NOT between frames.
4. **No per-frame padding**: Individual frames do NOT get padded to byte boundaries.

## Analysis by Library

### 1. HIGHDICOM (Python) — WRITING

**Files**: [sop.py:1237-1296](highdicom/src/highdicom/seg/sop.py#L1237-L1296), [sop.py:2111-2130](highdicom/src/highdicom/seg/sop.py#L2111-L2130)

**Approach**: Uses a `remainder_pixels` accumulator. For each frame, concatenates leftover pixels from the previous frame with the current frame's pixels, then takes the largest multiple-of-8 chunk for encoding, carrying the leftover forward. Calls `pydicom.pack_bits(planes, pad=False)`.

**Verdict: COMPLIANT** — Bits are correctly packed continuously across frame boundaries. The remainder mechanism ensures no per-frame padding.

**Bug found (minor)**: At [sop.py:1296](highdicom/src/highdicom/seg/sop.py#L1296), the trailing padding byte is `b'0'` (ASCII 0x30) instead of `b'\x00'` (null byte). While the DICOM standard says applications must not assume anything about padding bit contents, using a non-zero ASCII character is unconventional and could cause issues with strict parsers.

**Condition note**: At [sop.py:1242](highdicom/src/highdicom/seg/sop.py#L1242), the condition `(self.Rows * self.Columns) // 8 != 0` uses integer division (`//`) rather than modulo (`%`). This means the condition is True whenever Rows×Columns >= 8 (virtually always), so the continuous packing logic is effectively always used. The condition should probably be `% 8 != 0` for clarity, but as-is it doesn't cause a functional bug.

### 2. HIGHDICOM (Python) — READING

**Files**: [frame.py:441-446](highdicom/src/highdicom/frame.py#L441-L446), [io.py:586-597](highdicom/src/highdicom/io.py#L586-L597), [io.py:655-663](highdicom/src/highdicom/io.py#L655-L663), [image.py:2657-2666](highdicom/src/highdicom/image.py#L2657-L2666)

**Approach for frame extraction** (`image.py`):
- When `BitsAllocated == 1` and `n_pixels % 8 != 0`: computes byte offsets using `start = (frame_index * frame_length_bits) // 8` and `end = ((frame_index + 1) * frame_length_bits + 7) // 8`, correctly extracting the overlapping bytes from the continuous bitstream.

**Approach for frame decoding** (`frame.py`):
- Unpacks the raw bytes using `pydicom.unpack_bits()`, then computes a `pixel_offset` to handle the fact that the frame's first pixel may not be at bit 0 of the first byte: `pixel_offset = int(((index * n_pixels / 8) % 1) * 8)`.
- This correctly identifies how many bits into the first byte the frame data starts.

**Approach for `ImageFileReader`** (`io.py`):
- Offset table computed as `int(np.floor(i * n_pixels / 8))` for 1-bit data — correctly computing continuous byte offsets.
- Reads `_bytes_per_frame_uncompressed` = `n_pixels // 8 + (n_pixels % 8 > 0)` bytes — ceiling division, which may read one extra byte containing bits from the next frame's start. The `decode_frame` function then strips these extra bits using the pixel_offset logic.

**Verdict: COMPLIANT** — Reading correctly handles continuous bit packing across frames.

### 3. DCMJS (JavaScript) — WRITING

**Files**: [bitArray.js:28-59](dcmjs/src/bitArray.js#L28-L59), [Segmentation.js:112-141](dcmjs/src/derivations/Segmentation.js#L112-L141)

**Approach**: Allocates a single unpacked buffer for ALL frames (`Rows * Columns * NumberOfFrames` bytes), fills in frame data at linear byte offsets, then calls `BitArray.pack()` on the entire buffer at once.

`BitArray.pack()` uses a single linear index across all pixels: `bytePos = Math.floor(i / 8)`, `bitPixelValue = pixValue << (i % 8)`.

**Verdict: COMPLIANT** — Packing the entire multi-frame buffer at once inherently produces continuous bit packing.

### 4. DCMJS (JavaScript) — READING

**Files**: [Segmentation_4X.js:348,1091-1106](dcmjs/src/adapters/Cornerstone/Segmentation_4X.js#L1091-L1106), [bitArray.js:64-76](dcmjs/src/bitArray.js#L64-L76), [Segmentation_4X.js:626-631](dcmjs/src/adapters/Cornerstone/Segmentation_4X.js#L626-L631)

**Approach**: Calls `BitArray.unpack()` on the entire PixelData, producing a byte-per-pixel array of length `8 * ceil(totalPixels/8)`. Then accesses frame data using `frameSegment * sliceLength` where `sliceLength = Rows * Columns`.

**Verdict: COMPLIANT** — Unpacking the entire bitstream into a linear byte array and then indexing by pixel count (not byte count) correctly handles frames that don't start on byte boundaries.

### 5. DCMTK (C++) — WRITING

**Files**: [segutils.h:118-168](dcmtk/dcmseg/include/dcmtk/dcmseg/segutils.h#L118-L168), [segutils.cc:30-98](dcmtk/dcmseg/libsrc/segutils.cc#L30-L98)

**Approach**: Each frame is first packed individually via `packBinaryFrame()` (producing per-frame padded byte arrays). Then `concatBinaryFrames()` reassembles all frames into a single continuous bitstream using a persistent `bitIndex` counter across all frames.

When `frameBits % 8 == 0`, uses optimized `memcpy`. When `frameBits % 8 != 0`, does bit-by-bit copying with the continuous `bitIndex`.

Total size: `(totalBits + 7) / 8` — single rounding at the end.

**Verdict: COMPLIANT** — The two-step process (per-frame pack, then continuous concat) correctly produces a continuous bitstream.

### 6. DCMTK (C++) — READING

**Files**: [iodutil.cc:690-794](dcmtk/dcmiod/libsrc/iodutil.cc#L690-L794)

**Approach**: `extractBinaryFrames()` iterates over all bits in the continuous PixelData using a single loop with `bitsLeftInInputByte`, `bitsLeftInFrame`, and `bitsLeftInTargetByte` counters. When a frame boundary is reached (mid-byte), it resets frame-level counters while continuing to read from the same input byte position.

Each extracted frame gets its own byte array with per-frame padding (last byte padded with zeros). This is an internal representation detail — the frames were correctly split from the continuous stream.

**Verdict: COMPLIANT** — Reading correctly handles continuous bit packing.

### 7. DCMQI (C++) — Delegates to DCMTK

**Files**: [Itk2DicomConverter.cpp](dcmqi/libsrc/Itk2DicomConverter.cpp) (writing), [Dicom2ItkConverter.cpp:153-157](dcmqi/libsrc/Dicom2ItkConverter.cpp#L153-L157) (reading)

Uses DCMTK's `DcmSegUtils::packBinaryFrame()` / `DcmSegUtils::unpackBinaryFrame()` and `DcmSegmentation::addFrame()` / `DcmSegmentation::getFrame()`.

**Verdict: COMPLIANT** — Inherits DCMTK's correct behavior.

### 8. PIXELMED (Java) — WRITING

**Files**: [IndexedLabelMapToSegmentation.java:308-315](pixelmed/com/pixelmed/convert/IndexedLabelMapToSegmentation.java#L308-L315), [IndexedLabelMapToSegmentation.java:490-504](pixelmed/com/pixelmed/convert/IndexedLabelMapToSegmentation.java#L490-L504)

**Approach**: Allocates a single byte array for ALL frames. Uses `setBit()` with a global pixel offset: `pixelOffset = pixelsPerFrame * f + columns * r + c`. Size: `dstPixelCount/8` rounded up, then rounded up to even.

**Verdict: COMPLIANT** — Global pixel offset computation inherently produces continuous bit packing.

### 9. PIXELMED (Java) — READING

**Files**: [ShrinkSegmentationToBoundingBox.java:318-325](pixelmed/com/pixelmed/apps/ShrinkSegmentationToBoundingBox.java#L318-L325)

**Approach**: `getBit()` uses the same global pixel offset formula as `setBit()`.

**Verdict: COMPLIANT** — Symmetric with writing.

### 10. WEASIS (Java) — READING (via dcm4che3/weasis-dicom-tools)

**Files**:
- [DicomMediaIO.java:654-688](Weasis/weasis-dicom/weasis-dicom-codec/src/main/java/org/weasis/dicom/codec/DicomMediaIO.java#L654-L688) — frame reading entry point
- [SegSpecialElement.java](Weasis/weasis-dicom/weasis-dicom-codec/src/main/java/org/weasis/dicom/codec/SegSpecialElement.java) — segmentation metadata handling
- [PhotometricInterpretation.java:196-198](dcm4che/dcm4che-image/src/main/java/org/dcm4che3/image/PhotometricInterpretation.java#L196-L198) (dcm4che3) — frame byte length calculation
- [DicomImageReader.java:776-787](weasis-dicom-tools/weasis-dicom-tools/src/main/java/org/dcm4che3/img/DicomImageReader.java#L776-L787) (weasis-dicom-tools) — frame offset computation

**Architecture**: Weasis itself does NOT implement bit unpacking. It delegates to:
1. **dcm4che3** (via weasis-dicom-tools v5.34.1.2) for DICOM pixel data reading
2. **OpenCV** (via `Imgcodecs.dicomRawFileRead()`) for native image decoding

**Frame offset calculation** in `DicomImageReader.buildBulkDataStream()`:
```java
int frameLength = desc.getPhotometricInterpretation()
    .frameLength(desc.getColumns(), desc.getRows(), desc.getSamples(), desc.getBitsAllocated());
long[] offsets = {bulkData.offset() + (long) frameIndex * frameLength};
```

**`PhotometricInterpretation.frameLength()`** in dcm4che3:
```java
public int frameLength(int w, int h, int samples, int bitsAllocated) {
    return w * h * samples * bitsAllocated / 8;
}
```

**Verdict: NON-COMPLIANT** — Two bugs:

1. **Frame length truncation**: For BitsAllocated=1 with non-byte-aligned frames, `w * h * 1 / 8` uses integer division which **truncates** instead of ceiling. For 187x239 (44,693 pixels): `44693 / 8 = 5586` bytes, but the frame needs **5587 bytes** (`ceil(44693/8)`). This means 5 pixels are lost from each frame.

2. **Per-frame byte-aligned offsets**: Frame N starts at `frameIndex * frameLength`, which assumes each frame begins on a byte boundary. Per the DICOM standard, frame N should start at bit `N * rows * cols`, which may be mid-byte. For 187x239, frame 1 should start at bit 44,693 (byte 5586, bit 5), not at byte 5586 bit 0.

These two bugs compound across frames, producing the progressively-shifting artifact reported by the user.

## Summary

| Library | Language | Writing | Reading | Notes |
|---------|----------|---------|---------|-------|
| **highdicom** | Python | COMPLIANT | COMPLIANT | Minor bug: padding byte is `b'0'` (0x30) not `b'\x00'`; misleading condition uses `//` instead of `%` |
| **dcmjs** | JavaScript | COMPLIANT | COMPLIANT | Clean implementation, packs entire buffer at once |
| **DCMTK** | C++ | COMPLIANT | COMPLIANT | Two-step process (per-frame pack + concat) is correct (at current version) |
| **dcmqi** | C++ | COMPLIANT | COMPLIANT | Delegates to DCMTK |
| **pixelmed** | Java | COMPLIANT | COMPLIANT | Global offset formula is elegant and correct |
| **Weasis** | Java | N/A (viewer) | **NON-COMPLIANT** | dcm4che3's `frameLength()` truncates instead of ceiling; per-frame byte-aligned offsets |

## DCMTK Version History — Three Rounds of Bug Fixes

DCMTK's binary segmentation bit packing code has gone through **three rounds of bug fixes** spanning 2018-2024, with the fundamental approach being completely rewritten in 2024. The old approach used byte-level `memcpy` + whole-frame bit-shifting (a complex and error-prone technique), while the new approach uses clean bit-by-bit iteration.

### Timeline of DCMTK Bug Fixes

#### Round 1: June 2018 — Commit [`ba542273f`](https://github.com/DCMTK/dcmtk/commit/ba542273f) ("Fixed binary segmentations with rows*cols%8 != 0")

**Context**: Reported via [dcmqi issue #341](https://github.com/QIICR/dcmqi/issues/341) by Andrey Fedorov.

**Bug**: The original `shiftRight()` and `shiftLeft()` functions had **inverted shift directions**. The code confused the bit-level direction (LSB-first packing) with byte-level operations. Writing used `shiftRight` where it should have shifted left, and reading used `shiftLeft` where it should have shifted right.

**Fix**: Renamed to `alignFrameOnBitPosition()` / `alignFrameOnByteBoundary()` and swapped all `<<` with `>>` (and vice versa). Corrected the mask operation for clearing unused bits in the last byte.

**Remaining issues**: The fix introduced a **signed integer promotion bug** — the expression `(byte << n) >> n` (used to mask out unused bits) didn't work correctly because C integer promotion causes the left-shift result to remain in `int` space (32 bits), so the right-shift doesn't clear the bits that were shifted into higher positions. Also, the fundamental **frame length calculation** issue remained.

#### Round 2: March 2022 — Commits [`95335baed`](https://github.com/DCMTK/dcmtk/commit/95335baed) and [`1fd08a5fa`](https://github.com/DCMTK/dcmtk/commit/1fd08a5fa) ("Fix some binary segmentation concatenations")

**Bug**: Signed integer promotion issue from the 2018 fix. When `Uint8` values were shifted left, they were promoted to `signed int`, and subsequent right-shifts could produce incorrect results due to sign extension. For example, `(byte << 4) >> 4` doesn't clear the top 4 bits when the intermediate value is wider than 8 bits.

**Fix**: Added explicit `OFstatic_cast(unsigned char, ...)` casts to truncate shifted values back to 8 bits before right-shifting. This was applied to `extractBinaryFrames`, `concatFrames`, `alignFrameOnBitPosition`, and `alignFrameOnByteBoundary`.

**Remaining issues**: The fundamental **frame byte length calculation** issue remained. The code computed `frameLengthBytes = bitsPerFrame/8 + 1` (when not byte-aligned), but in certain configurations, accessing the continuous bitstream requires reading `bitsPerFrame/8 + 2` bytes. This caused the byte-shifting to miss data in the boundary bytes, leading to corruption that accumulated across frames.

#### Round 3: December 2024 — Commit [`6245f9cc9`](https://github.com/DCMTK/dcmtk/commit/6245f9cc9) ("Fix binary segmentations with certain dimensions")

**Context**: Reported as [DCMTK issue #1132](https://support.dcmtk.org/redmine/issues/1132) by Melanie Michels.

**Bug**: The byte-shifting approach in `concatFrames` and `extractBinaryFrames` had a **frame length calculation error**. When frame bits span a byte boundary at the end and the shift operation needs to read/write one more byte than `ceil(bitsPerFrame/8)`, the code would fail to process all bits correctly. The 2024 commit message notes that the bugs "mostly cancelled out during DCMTK write-then-read cycles, masking the issue until external software encountered the malformed data."

This is a key insight: **the writing and reading bugs were symmetric**, so DCMTK could read back its own malformed output. But other tools that implemented correct DICOM reading would see the corruption.

**Fix**: Complete rewrite of all bit-packing functions:
- `concatFrames` -> `concatBinaryFrames`: Now uses a single continuous `bitIndex` counter, iterating bit-by-bit across all frames. With a fast-path `memcpy` optimization when `frameBits % 8 == 0`.
- `extractBinaryFrames` (in `DcmIODUtil`): Now uses `bitsLeftInInputByte`, `bitsLeftInFrame`, `bitsLeftInTargetByte` counters for clean bit-by-bit extraction.
- `packBinaryFrame` and `unpackBinaryFrame`: Rewritten as templates with clear `byteIndex = i/8`, `bitIndex = i%8` logic.
- Old `alignFrameOnBitPosition` / `alignFrameOnByteBoundary` helper functions were **removed entirely**.
- Added comprehensive randomized tests (1000 iterations with random dimensions) for pack, unpack, and concat operations.

### Impact on 3D Slicer

3D Slicer uses dcmqi to load DICOM SEG, and dcmqi depends on DCMTK. The dcmqi in this repo pins to DCMTK commit `3e85b37` (post-December 2025, i.e., after all three fixes). However, **released versions of 3D Slicer** at the time of the highdicom issue #393 would have used an older DCMTK that contained the frame-length calculation bug (fixed only in December 2024).

The **writing-side bug in old DCMTK** meant that segmentations created by dcmqi (using old DCMTK) with non-byte-aligned dimensions produced **malformed PixelData**. This malformed data was readable by DCMTK/dcmqi itself (bugs cancelled out symmetrically) but incorrectly displayed by other viewers implementing correct DICOM reading.

However, the segmentations in highdicom issue #393 were **created by highdicom**, not dcmqi. So the writing was correct. The issue was in the **reading/viewing tools** — specifically 3D Slicer (via old DCMTK) and Weasis (via dcm4che3).

## Minor Issues Found

1. **highdicom** [sop.py:1296](highdicom/src/highdicom/seg/sop.py#L1296): Padding byte `b'0'` should be `b'\x00'`. The current code appends ASCII character '0' (0x30) instead of a null byte. While the DICOM standard says receivers must not assume padding bit contents, this is clearly a typo/bug.
2. **highdicom** [sop.py:1242](highdicom/src/highdicom/seg/sop.py#L1242): Condition `(self.Rows * self.Columns) // 8 != 0` uses integer division (`//`) where modulo (`%`) was likely intended. Current behavior: enters continuous-packing branch for any image >= 8 pixels (always). Intended behavior: enter it only when frame size is not byte-aligned. Functionally equivalent for all realistic images, but misleading.

## Conclusions

1. **All five libraries (at the versions in this repo) correctly implement continuous bit packing on the writing side** per the DICOM standard. The reading side is also correct for highdicom, dcmjs, DCMTK (current version), dcmqi, and pixelmed.

2. **Weasis (via dcm4che3) has a bug in its reading-side frame offset calculation.** The `PhotometricInterpretation.frameLength()` method in dcm4che3 uses `w * h * samples * bitsAllocated / 8` (truncating integer division) instead of ceiling division for 1-bit data. Combined with per-frame byte-aligned offset calculation (`frameIndex * frameLength`), this produces incorrect frame extraction for binary segmentations with non-byte-aligned dimensions. This is likely the cause of the Weasis rendering issues reported in issue #393.

3. **DCMTK had a persistent bug** with binary segmentation bit packing for non-byte-aligned frames, requiring **three rounds of fixes** spanning 2018-2024:
   - **2018**: Inverted shift directions fixed (reported via [dcmqi #341](https://github.com/QIICR/dcmqi/issues/341))
   - **2022**: Signed integer promotion bugs fixed
   - **2024**: Complete rewrite replacing the fragile byte-shift approach with clean bit-by-bit iteration (closes [DCMTK #1132](https://support.dcmtk.org/redmine/issues/1132)). The old approach had a **frame byte-length calculation error** where the byte-shifting operations sometimes needed to access more bytes than were allocated, causing data corruption that accumulated across frames. Critically, the writing and reading bugs were **symmetric**, so DCMTK could read its own malformed output correctly, masking the issue.

4. **The 2018 fix was partially correct but not sufficient.** It fixed the shift direction issue but introduced signed-shift bugs and didn't address the frame byte-length calculation issue. The 2022 fix addressed signedness but not the frame-length issue. Only the 2024 complete rewrite resolved all issues.

5. **The issue in 3D Slicer** (which uses dcmqi -> DCMTK for reading DICOM SEG) was caused by DCMTK's reading-side bug (pre-December 2024 version). Even though highdicom correctly wrote the continuous bitstream, DCMTK's `extractBinaryFrames` would incorrectly unpack frames when dimensions were not byte-aligned, producing the progressively-shifted frames the user reported.

6. **OHIF renders correctly** because it uses dcmjs, which has a clean implementation that unpacks the entire bitstream at once and accesses frames by pixel index (not byte index), inherently handling non-byte-aligned frame boundaries correctly.

7. **dcmjs and pixelmed** never had these issues because they use fundamentally simpler approaches:
   - **dcmjs**: Packs/unpacks the entire multi-frame buffer at once using a single linear pixel index — no per-frame byte-shifting needed.
   - **pixelmed**: Uses a global pixel offset formula (`pixelsPerFrame * f + columns * r + c`) to address bits directly — no byte-level operations needed.

## How This Report Was Created

This report was produced using [Claude Code](https://docs.anthropic.com/en/docs/claude-code), Anthropic's agentic coding tool, powered by Claude Opus. The investigation was conducted through iterative source code analysis across the five library submodules in this repository.

**Methodology:**

1. **DICOM standard review**: The relevant section of the DICOM standard ([Part 5, Section 8.1.1](https://dicom.nema.org/medical/dicom/current/output/chtml/part05/chapter_8.html#sect_8.1.1)) was fetched and analyzed to establish the normative requirements for 1-bit pixel data encoding.

2. **Source code analysis**: For each library, the writing and reading code paths for binary segmentation pixel data were traced by reading source files directly from the submodules. Key functions were identified by searching for bit packing, frame extraction, and pixel data assembly operations.

3. **Git archaeology (DCMTK)**: DCMTK's bug fix history was reconstructed by examining git commits (`ba542273f`, `95335baed`, `1fd08a5fa`, `6245f9cc9`) and their associated issue reports ([dcmqi #341](https://github.com/QIICR/dcmqi/issues/341), [DCMTK #1132](https://support.dcmtk.org/redmine/issues/1132)). Old and new versions of the bit packing code were compared to understand the evolution of bugs and fixes.

4. **External dependency analysis (Weasis)**: Since Weasis delegates pixel reading to external libraries (dcm4che3 and weasis-dicom-tools), the relevant source files were fetched from their GitHub repositories to trace the frame offset calculation and identify the truncating integer division bug.

5. **Cross-referencing with the reported issue**: Findings were validated against the specific symptoms described in [highdicom issue #393](https://github.com/ImagingDataCommons/highdicom/issues/393) — progressively shifting frames with 187x239 dimensions — to confirm that the identified bugs would produce the observed behavior.
