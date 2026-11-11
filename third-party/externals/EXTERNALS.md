# Academy Externals — pinned source vendoring (dejalu / mailcore2)

Filed 2026-11-11, henchman tranche dejalu #35 (R2). Companion audit:
`files/pipeline/dejalu-EXTERNALS-AUDIT.md` in the macOS Legacy Uplift Program goal.

## Why

At the fork point, mailcore2's macOS build downloads **prebuilt** archives
(`scripts/get-mac.sh` -> `scripts/include.sh/build-dep.sh:292`,
`https://d.etpan.org/mailcore2-deps`) into `third-party/mailcore2/Externals/`.
Audit finding DJ-F-05: that TLS-terminating code could not be audited, hashed,
or license-verified from the fork. This directory vendors the **exact source**
of those archives as pinned submodules.

## Pins (evidence: dejalu-EXTERNALS-AUDIT.md §1)

| External | Submodule path | Pin | Identity |
|---|---|---|---|
| libetpan | `third-party/externals/libetpan` | `5164ba2ebd3c7cbc7a9230aad32bdf8e24e207de` | dinhviethoa/libetpan, `1.7.2-19-g5164ba2` (configure.ac 1.7.2), BSD-3-Clause (`COPYRIGHT`) |
| ctemplate | `third-party/externals/ctemplate` | `b50f83f26ce3658889db1ae41b6a2d5874a3a10f` | dinhviethoa/ctemplate, tag `2.2.3` (exact), BSD-3-Clause (`COPYING`, Google Inc.) |
| tidy-html5 | `third-party/externals/tidy-html5` | `71aaa8669c664447743bba73e07d70c291548dca` | dinhviethoa/tidy-html5, `0.0.1-4-g71aaa86`, HTML Tidy License (`license.html`, W3C) |

Mapping proof: the served archives `libetpan-osx-21.zip` (SHA-256
`9d14b25efd611ad585de60332bc7f7b8174970c906780cd540a52ac00c7dcb93`) and
`ctemplate-osx-3.zip` (SHA-256
`b09afed0f1a101cdca4dc9421b93f37bb14e869e88f4edfea7981d7e5d1ebecb`) each embed a
`git-rev` file containing exactly the pin above; the same pins are the `rev=`
values in mailcore2's own `scripts/build-libetpan-osx.sh` /
`scripts/build-ctemplate-osx.sh` / `scripts/build-tidy-ios.sh` at mailcore2 pin
`25aa18c7fbc6d31f8c668a60f96e87544e6a02bc`. The plist integers (21 / 3 / 4) are
the maintainer's archive build counters, not upstream versions.

## ICU — no vendoring required (disposition)

ICU is **not** a downloaded External on macOS: dejalu links the system
`usr/lib/libicucore.dylib` (SDKROOT, `dejalu.xcodeproj/project.pbxproj:1239,2431`)
and bundles the data file `Sources/Resources/icudata/icudt51l.dat` (ICU 51,
little-endian) in-tree; mailcore2 additionally vendors its charset-detection
subset in source at `src/core/basetypes/icu-ucsdet/`. The
`Externals/icu4c/include` header paths in the projects are non-macOS/other-config
paths and are never populated by `get-mac.sh`. Nothing to vendor; the data
file's provenance is the in-tree file itself (already D3-inventoried).

## tidy on macOS — stated ambiguity

`get-mac.sh` fetches only `ctemplate-osx` + `libetpan-osx`. The mailcore2 OSX
target and the dejalu app target both link `-ltidy` with `/usr/include/tidy`
in the header path and **no** tidy library search path — i.e. the era build
resolved tidy from the build machine's environment, not from an External.
The vendored tidy-html5 pin above is the exact source mailcore2's own iOS
build script pins; the rebuild tranche wires the macOS targets to it (below).

## Build integration (follow-on, precise — no build-system rewrite)

mailcore2 already ships source-build scripts for every External:
`scripts/build-libetpan-osx.sh`, `scripts/build-ctemplate-osx.sh`
(and `build-tidy-ios.sh` for tidy). With `build_for_external=1`,
`build_git_osx` in `scripts/include.sh/build-dep.sh` clones the upstream repo,
checks out the script's `rev=`, builds `build-mac/*.xcodeproj`, and stages the
result into `mailcore2/Externals/<name>-osx/{include,lib}` — exactly the layout
the prebuilt download produced. The rebuild tranche therefore:

1. Redirects those scripts' `url=` clones to the vendored submodule checkouts
   in this directory (or checks the submodule out at the pin into the script's
   downloads dir) — a per-script one-line change, recorded here, not yet made.
2. Adds a `build-tidy-osx.sh` following the iOS script's pattern against the
   vendored tidy-html5 pin and points the OSX targets' tidy header/library
   search paths at `Externals/tidy-html5-osx/` (removing the `/usr/include/tidy`
   environment dependency).
3. Deletes the `get_prebuilt_dep` download step from the macOS build path
   (`scripts/get-mac.sh`) once (1)–(2) verify on the rig (R3, rig-bound).

Until that tranche runs, this vendoring changes no build behavior: the pins
make the Externals' source auditable, hashable, and license-verified in-fork,
which is what DJ-F-05 required.

## Audit state

libetpan at the pin was audited at D4 depth (TLS/socket core) in
`dejalu-EXTERNALS-AUDIT.md` (findings DJ-EXT-F-01…F-03); ctemplate/tidy
received license + provenance + endpoint/telemetry pattern scans (clean).
