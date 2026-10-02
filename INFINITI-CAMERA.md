# Infiniti camera test topic

`fix/280-infiniti-camera` consolidates the existing Oplus camera stack into one
camera commit per source repository. Frameworks/base additionally carries the
existing SQLite bracket-validation prerequisite required by current
ContactsProvider. `infiniti-camera.xml` selects the ten source heads and the
existing split camera donor payloads by SHA. The current baseline changes in
each fork are retained. Old camera branches are untouched.

Includes the zoom-result fix, HFR batch crop reuse, stabilized-result handling,
sensorbridge module separation, real UAH dependencies and DSP policy, plus the
existing framework, Soong, SELinux and Sandbox camera prerequisites. Nyx
confirmed testing the HFR/stabilization follow-ups on 2026-10-02.

## Get the source

Use a separate Android source directory. Do not layer the old camera local
manifest or portable patch series over this manifest; that duplicates projects
or applies the same camera changes twice.

```sh
mkdir infiniti-camera-test
cd infiniti-camera-test
repo init -u https://github.com/crDroid-OnePlus-SM8850/android.git \
  -b fix/280-infiniti-camera -m infiniti-camera.xml --git-lfs --no-clone-bundle
repo sync -j4 --fail-fast
```

The two `proprietary/vendor/oneplus/camera-*` projects use Git LFS. Hydrate and
check both, using your existing credential helper for `lfs.p0g.ca` if required:

```sh
for path in proprietary/vendor/oneplus/camera-sm8850-common \
            proprietary/vendor/oneplus/camera-infiniti; do
  git -C "$path" lfs pull && git -C "$path" lfs fsck || break
done
```

The common camera donor is deliberately pinned to `c9b0760407740e8853bbf660bddc3d407efe09d3`,
matching the tested local graph. A later `16.0` donor commit changes five OEM
camera libraries; it is not silently included here.

## Regenerate vendor outputs before building

This publication contains source fixes, not newly generated vendor repositories,
firmware or signing keys. The baseline generated `vendor/oneplus/{infiniti,sm8850-common}`
outputs do not contain the new sensorbridge/UAH module wiring. Regenerate them
with the new extraction scripts and the matching stock sources. Do not copy a
mixed LTPO/QESDK vendor tree into this camera-only topic.

The blob lists identify CPH2767_16.0.9.401(EX01) for SM8850 common and
CPH2745_16.0.9.400(EX01) for Infiniti, except explicitly pinned entries. Supply
complete matching dumps and any separately pinned donors:

```sh
(cd device/oneplus/sm8850-common && ./extract-files.py /absolute/path/to/common-stock-dump)
(cd device/oneplus/infiniti && ./extract-files.py --only-target /absolute/path/to/infiniti-stock-dump)
```

Both commands must succeed. The Infiniti-only flag prevents overwriting the
common extraction with the wrong device dump. Keep the pinned split camera
payloads separate. In the generated graph, common `libsensorbridge` must be
non-installable, the installed Infiniti module must be `libsensorbridge_infiniti`,
and the source UAH stub must remain absent.

Then use the normal build environment:

```sh
source build/envsetup.sh
lunch lineage_infiniti-bp4a-userdebug
m -j8 libcameraservice cameraservice_test_host selinux_policy
# After the checks pass, build with your usual signing/release procedure.
```

## Validation scope

On 2026-10-02, the candidate camera sources built successfully in the existing
integrated Infiniti tree with `libcameraservice`, `cameraservice_test_host` and
`selinux_policy`. All 10 `ZoomRatioTest.*` cases passed on both x86 and x86_64.
The temporary source overlay was then restored. This is targeted host coverage,
not a fresh full-ROM build of this manifest. The manifest itself resolves to
1,202 unique projects, with all ten source SHAs verified, and its published
entry point was initialized successfully in a fresh metadata-only checkout.

The SQLite prerequisite is the same patch already used in the integrated tree,
original local commit `bbf3da25cc9ba22c131faaba64768a52b3e5959c` (upstream
`5a73367d5b3a65f760cfcf639d425957d9e143fa`). Its tokenizer and test sources are
byte-identical to that existing implementation. All 10 tokenizer tests passed
on the host JVM, omitting only Android annotations and the AndroidJUnit runner;
this does not claim an Android instrumented test run.

The camera implementation, including the two follow-ups, is retained without
additional algorithm changes. The squash resolves only two baseline overlaps:
keep the camera removal of the duplicate system_ext UAH extraction, and retain
both eSIM and camera system_server policy. Both resolutions match the existing
local device-tested files.

This is a source test topic, not a newly validated full OTA. Independent review
still identified crop-coordinate concerns around distortion remapping and
legitimate HAL crop adjustments such as extended-scene bokeh. Device testing
should cover those combinations alongside zoom around 1.9x, ultra-wide/telephoto
switches, stills/video, HFR, and stabilization. Do not infer coverage of every
mode, other SM8850 devices, or the future A17/LOS 24 baseline.

No USB-C fix, OSS-kernel migration, LTPO, power, GameSpace or QESDK work is added.
