# WhateverXe patch — Tiger Lake 80 EU with 6 L3 banks

A minimal source patch that lets [WhateverXe](https://github.com/b00t0x/WhateverXe) accept Tiger Lake GT2 parts whose fuse layout is **80 EU + 6 L3 banks**. The current shape table rejects that combination, so the accelerator never probes and Metal stays unavailable even though the framebuffer works.

Upstream report: [b00t0x/WhateverXe#10](https://github.com/b00t0x/WhateverXe/issues/10)

> Unofficial community patch. Not affiliated with the WhateverXe maintainer. Use at your own risk.

## Who is affected

Tiger Lake-U parts where the GT fuse readout decodes to **dss = 5, eus = 80, l3Banks = 6** — for example some i5-1135G7 units.

Quick check. If `TGLGT2Qualified` is `No` while the framebuffer already drives the panel, you are likely hitting this:

```sh
ioreg -l -w0 | grep -o '"TGLGT2Qualified" = [A-Za-z]*'
ioreg -rc AppleIntelTGLGraphics | wc -l            # 0 = accelerator never instantiated
```

Symptoms:

- internal display works, `FB0: Complete modeset` in the log
- no IOGPU device; runningboardd logs `setGPURole:pid: failed due to nil IOGPU device ref`
- `MTLCreateSystemDefaultDevice()` returns nil
- `system_profiler SPDisplaysDataType` lists the GPU but no Metal line

## Root cause

`WhateverXe/kern_xelp_gt_resources.hpp` -> `decodeTGLGT2()` accepts only:

```cpp
(decoded.dss == 3 && decoded.eus == 48 && decoded.l3Banks == 6) ||
((decoded.dss == 5 || decoded.dss == 6) && decoded.eus == 16 * decoded.dss && decoded.l3Banks == 8)
```

An 80 EU part with 6 L3 banks satisfies neither branch. `tglGT2Ready` stays false, `XE::wrapTGLAcceleratorProbe()` returns `nullptr` at its first gate, and `IntelAccelerator` is never created.

Measured fuse values on an affected machine (ASUS VivoBook X515EA, i5-1135G7):

```
fuse0 = 0x00000001   slice   = 1
fuse1 = 0x0000001F   dss     = 5    (popcount)
fuse2 = 0x00000000   perDSS  = 16   -> eus = 80
fuse3 = 0x00000088   banks   = 6    (8 - popcount(0x88))
fuse4 = 0x000E00FA   video = 5, enhance = 1
status = 4 (ProfileMismatch)
```

`decodeADLPGT()` passes all six checks; only the GT2 shape filter fails.

## The patch

One line, in `WhateverXe/kern_xelp_gt_resources.hpp`:

```diff
-    if(!((decoded.dss==3 && decoded.eus==48 && decoded.l3Banks==6) ||
-         ((decoded.dss==5 || decoded.dss==6) && decoded.eus==16*decoded.dss && decoded.l3Banks==8)))return false;
+    if(!((decoded.dss==3 && decoded.eus==48 && decoded.l3Banks==6) ||
+         ((decoded.dss==5 || decoded.dss==6) && decoded.eus==16*decoded.dss &&
+          (decoded.l3Banks==8 || decoded.l3Banks==6))))return false;
```

The decoded `l3Banks` is then propagated into the URB table (`urbReplaceAlderLakeN[17]`), matching the physical L3 layout instead of assuming 8 banks.

## Build

Requirements:

- macOS with Xcode Command Line Tools (`xcode-select --install`)
- `python3` (system Python works)

```sh
chmod +x build.sh
./build.sh
```

The script downloads the pinned WEX source, MacKernelSDK and Lilu headers, applies the patch, builds a release kext and prints its path. Nothing is installed automatically.

Override the version if needed:

```sh
WEX_TAG=0.7.0 ./build.sh
```

## Install

```sh
sudo diskutil mount disk0s1
cp -R build/WhateverXe.kext /Volumes/EFI/EFI/OC/Kexts/
```

Keep a copy of the previous kext for rollback.

## Verify

After reboot:

```sh
ioreg -l -w0 | grep -o '"TGLGT2Qualified" = [A-Za-z]*'   # expect Yes
ioreg -rc AppleIntelTGLGraphics | wc -l                  # expect >= 1
system_profiler SPDisplaysDataType | grep -iE "Chipset|Metal"
```

Success looks like `Metal Support: Metal 3` plus an `IntelAccelerator` instance.

## Rollback

Restore the previous `WhateverXe.kext` in `EFI/OC/Kexts` and reboot. If the system no longer boots, do it from another OS — the ESP is plain FAT32.

## Notes and limitations

- Tested on macOS 15.8.2 (24H232) with WhateverXe 0.7.0 and WhateverXe Graphics 0.4.0.
- If `TGLGT2Qualified` becomes `Yes` but Metal is still missing, the next gate is `tglAcceleratorReady` in `kern_xe.cpp`, which ANDs a dozen other readiness flags.
- URB sizing for 6-bank parts is not validated upstream. Watch for rendering corruption or hangs under sustained GPU load.
- Do not combine with WhateverGreen.

## License

The patch modifies WhateverXe, distributed under BSD-3-Clause. Redistribution must keep the original copyright notice. See `LICENSE-NOTICE.md`.
