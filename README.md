# Chromium NVIDIA CUVID Hardware Decoding

## Project overview

This project modifies Chromium to enable NVIDIA GPU hardware decoding of H.264 and HEVC video streams using FFmpeg's CUVID decoders on Linux x86-64.

The target environment is:

- **Chromium:** 159.0.8143.0
- **Operating system:** Fedora 41
- **Architecture:** Linux x86-64
- **GPU:** NVIDIA GeForce GTX 1660 (6 GB)
- **NVIDIA driver:** 580.105.08
- **CUDA:** 13.0
- **FFmpeg:** Chromium's bundled FFmpeg
- **Build system:** GN / Ninja

The primary motivation is smooth playback of high-resolution UniFi Protect camera streams, particularly HEVC streams that were not supported by the original Chromium FFmpeg configuration.

The implementation explicitly selects NVIDIA's CUVID decoders for H.264 and HEVC while retaining Chromium's existing software decoders for other codecs.

## Current status

### Working

- Chromium FFmpeg configured with NVIDIA FFmpeg/CUVID support.
- H.264 and HEVC CUVID decoder implementations compiled into Chromium.
- CUVID decoders registered in FFmpeg's generated codec list.
- HEVC decoder enabled in Chromium's FFmpeg integration.
- CUVID decoder selection implemented in `FFmpegVideoDecoder`.
- NV12 and P010 pixel formats mapped into Chromium's video frame formats.
- CUVID successfully initialises and decodes video when Chromium runs with `--no-sandbox`.
- UniFi Protect streams tested successfully with both H.264 and HEVC.
- NVIDIA GPU decoder utilisation observed during playback.

### Outstanding

Normal sandboxed playback currently fails because the renderer's seccomp policy prevents the CUVID loader from opening NVIDIA driver libraries.

The failure has been traced to denied `openat()` calls, not missing libraries or missing FFmpeg decoder registration.

A proposed fix preloads the NVIDIA driver libraries in the renderer immediately before sandbox initialisation. This change still needs to be built and tested.

The eventual deliverable will be a consolidated `.diff` containing the verified project modifications.

---

## 1. Build configuration

The Chromium checkout is located at:

```text
~/git/chromium/src
```

The build output directory is:

```text
out/HEVC
```

Relevant GN arguments:

```gn
is_debug = false
is_component_build = false
is_official_build = true

proprietary_codecs = true
ffmpeg_branding = "Chrome"

enable_platform_hevc = true
enable_hevc_parser_and_hw_decoder = true

use_vaapi = true
chrome_pgo_phase = 0

target_os = "linux"
target_cpu = "x64"

platform_has_optional_hevc_decode_support = false
```

Build Chromium with:

```bash
autoninja -j18 -C out/HEVC chrome
```

The build uses Chromium's bundled FFmpeg rather than a system FFmpeg installation.

## 2. Enabling CUVID in bundled FFmpeg

### FFmpeg configuration

File:

```text
media/ffmpeg/scripts/build_ffmpeg.py
```

The Chrome configuration has been extended to enable NVIDIA's FFmpeg hardware decoding support on Linux x86-64:

```python
# NVIDIA CUVID hardware decoding for Linux x64.
if target_os == 'linux' and target_arch == 'x64':
    configure_flags['Chrome'].extend(
        [
            '--enable-ffnvcodec',
            '--enable-cuvid',
            '--enable-decoder=h264_cuvid,hevc_cuvid',
        ]
    )
```

The existing software decoder configuration was also extended to include HEVC:

```text
--enable-decoder=aac,h264,hevc
```

The resulting FFmpeg configuration must contain:

```c
#define CONFIG_CUVID 1
#define CONFIG_FFNVCODEC 1
```

And `config_components.h` must contain:

```c
#define CONFIG_H264_CUVID_DECODER 1
#define CONFIG_HEVC_CUVID_DECODER 1
```

### FFmpeg headers

The NVIDIA FFmpeg headers are provided by the `ffnvcodec` development package.

On the build system:

```bash
pkg-config --modversion ffnvcodec
pkg-config --cflags --libs ffnvcodec
```

The required headers include:

```text
ffnvcodec/dynlink_loader.h
ffnvcodec/dynlink_cuda.h
ffnvcodec/dynlink_cuviddec.h
ffnvcodec/dynlink_nvcuvid.h
```

The headers have been staged under:

```text
third_party/ffmpeg/compat/cuda/include
```

The FFmpeg build must include this directory when compiling the CUVID implementation.

### FFmpeg configuration generation

The Chromium FFmpeg configuration script's Linux distribution detection does not work directly on this Fedora environment.

The following Python wrapper overrides the detected distribution to allow configuration generation:

```bash
cd ~/git/chromium/src

export PATH="$HOME/git/chromium/src/third_party/llvm-build/Release+Asserts/bin:$PATH"

python3 - <<'PY'
import sys
import runpy

sys.path.insert(0, 'media/ffmpeg/scripts')

from robo_lib import config, packages

config.GetLinuxFlavor = lambda: packages.OsFlavor.Arch

sys.argv = [
    'build_ffmpeg.py',
    'linux',
    'x64',
    '--branding=Chrome',
    '--config-only',
]

runpy.run_path(
    'media/ffmpeg/scripts/build_ffmpeg.py',
    run_name='__main__'
)
PY
```

Remove `--config-only` when performing the actual FFmpeg build.

## 3. FFmpeg decoder registration

Enabling the decoder configuration alone is insufficient.

Chromium's bundled FFmpeg uses generated codec registration files. The CUVID decoders must be included in the generated codec list.

The generated file:

```text
third_party/ffmpeg/build.x64.linux/Chrome/libavcodec/codec_list.c
```

must contain:

```c
&ff_h264_cuvid_decoder,
&ff_hevc_cuvid_decoder,
```

The checked-in configuration copy must be synchronised:

```bash
cp -v \
  third_party/ffmpeg/build.x64.linux/Chrome/libavcodec/codec_list.c \
  third_party/ffmpeg/chromium/config/Chrome/linux/x64/libavcodec/codec_list.c
```

The corresponding decoder declarations in `allcodecs.c` are:

```c
extern const FFCodec ff_h264_cuvid_decoder;
extern const FFCodec ff_hevc_cuvid_decoder;
```

The FFmpeg generated GN file must also include:

```text
libavcodec/cuviddec.c
```

### Bitstream filter registration

The HEVC CUVID decoder declares the following bitstream filter:

```c
DEFINE_CUVID_CODEC(hevc, HEVC, "hevc_mp4toannexb")
```

FFmpeg must therefore have the HEVC MP4-to-Annex-B bitstream filter registered.

The generated file:

```text
third_party/ffmpeg/build.x64.linux/Chrome/libavcodec/bsf_list.c
```

must contain the HEVC filter registration.

Synchronise it with:

```bash
cp -v \
  third_party/ffmpeg/build.x64.linux/Chrome/libavcodec/bsf_list.c \
  third_party/ffmpeg/chromium/config/Chrome/linux/x64/libavcodec/bsf_list.c
```

Without this entry, FFmpeg's decoder pre-initialisation fails with:

```text
AVERROR_BUG (-558323010)
```

This was an important distinction: the decoder itself was compiled and registered, but its required bitstream filter was missing from the checked-in generated configuration.

## 4. Chromium FFmpeg integration

### Decoder whitelist

File:

```text
media/ffmpeg/ffmpeg_common.cc
```

The permitted decoder list has been extended:

```cpp
return "h264,hevc,h264_cuvid,hevc_cuvid";
```

This allows Chromium's FFmpeg integration to use the CUVID decoders.

### Explicit decoder selection

File:

```text
media/filters/ffmpeg_video_decoder.cc
```

`FFmpegVideoDecoder::ConfigureDecoder()` explicitly selects the CUVID decoder for H.264 and HEVC:

```cpp
const AVCodec* codec = nullptr;

if (codec_context_->codec_id == AV_CODEC_ID_H264) {
  codec = avcodec_find_decoder_by_name("h264_cuvid");
} else if (codec_context_->codec_id == AV_CODEC_ID_HEVC) {
  codec = avcodec_find_decoder_by_name("hevc_cuvid");
} else {
  codec = avcodec_find_decoder(codec_context_->codec_id);
}
```

This deliberately leaves all other codecs on their normal FFmpeg decoder-selection path.

The implementation uses the CUVID decoders rather than the `h264_nvdec` / `hevc_nvdec` hardware-acceleration wrappers.

### HEVC support

Chromium's FFmpeg integration originally rejected HEVC in the relevant decoder support checks.

`FFmpegVideoDecoder::IsCodecSupported()` has been extended:

```cpp
bool FFmpegVideoDecoder::IsCodecSupported(VideoCodec codec) {
#if BUILDFLAG(ENABLE_PLATFORM_HEVC)
  if (codec == VideoCodec::kHEVC) {
    return true;
  }
#endif

  // We only build support for H.264.
  return codec == VideoCodec::kH264 && IsDecoderBuiltInVideoCodec(codec);
}
```

The required build flags header is included:

```cpp
#include "media/media_buildflags.h"
```

HEVC was also removed from the unsupported codec group in `GetFFmpegVideoDecoderThreadCount()`.

### Pixel formats

CUVID can return NV12 and P010 frames.

File:

```text
media/ffmpeg/ffmpeg_common.cc
```

The pixel-format conversion function includes:

```cpp
case AV_PIX_FMT_NV12:
  return PIXEL_FORMAT_NV12;

case AV_PIX_FMT_P010LE:
  return PIXEL_FORMAT_P010LE;
```

File:

```text
media/filters/ffmpeg_video_decoder.cc
```

The frame validation logic must account for biplanar formats. NV12 and P010 use Y and interleaved UV planes, so they do not provide a separate V plane.

The current validation is:

```cpp
const bool is_biplanar =
    frame->format == AV_PIX_FMT_NV12 ||
    frame->format == AV_PIX_FMT_P010LE;

if (!frame->data[VideoFrame::Plane::kY] ||
    !frame->data[VideoFrame::Plane::kU] ||
    (!is_biplanar && !frame->data[VideoFrame::Plane::kV])) {
  DLOG(ERROR) << "Video frame was produced yet has invalid frame data.";
  return false;
}
```

The `GetVideoBuffer()` DCHECK was also extended to accept `PIXEL_FORMAT_NV12` and `PIXEL_FORMAT_P010LE`.

## 5. CFI configuration

The CUVID initialisation path initially crashed with SIGILL inside:

```text
cuda_device_create()
```

GDB identified the trap as:

```text
__clang_trap_msg$Undefined Behavior Sanitizer$Control flow integrity check failed
```

The FFmpeg object compilation included indirect-call CFI instrumentation.

File:

```text
third_party/ffmpeg/BUILD.gn
```

The `ffmpeg_internal` target now disables indirect-call CFI on Linux:

```gn
if (is_linux) {
  configs += [ "//build/config/sanitizers:cfi_icall_disable" ]
}
```

This configuration must be placed after the target's `cflags = []` / `ldflags = []` initialisation and alongside the target's other configuration additions.

The resulting compile command includes both:

```text
-fsanitize=cfi-icall
-fno-sanitize=cfi-icall
```

The latter takes precedence.

This resolved the observed CFI trap. No further CFI changes should be made without new evidence of a separate failure.

## 6. NVIDIA driver loading and Chromium sandbox

### NVIDIA libraries

The system provides:

```text
libnvcuvid.so.1
libcuda.so.1
```

Both are available through the system dynamic linker.

The CUVID FFmpeg loader dynamically loads NVIDIA libraries using `dlopen()`.

### Observed sandbox failure

With Chromium running without `--no-sandbox`, the CUVID loader reports:

```text
CUVID DEBUG: loading library libnvcuvid.so.1
CUVID DEBUG: dlopen returned (nil)
CUVID DEBUG: dlopen error: libnvcuvid.so.1:
cannot open shared object file: Operation not permitted
```

The failure was investigated with:

```bash
strace -f -e trace=openat,openat2 \
  -o /tmp/chromium-open.log \
  ./out/HEVC/chrome \
    --user-data-dir=/tmp/chromium-cuvid-test
```

The relevant renderer process calls show:

```text
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = -1 EPERM

openat(AT_FDCWD, "/lib64/libnvcuvid.so.1",
       O_RDONLY|O_CLOEXEC) = -1 EPERM
```

Equivalent attempts against the glibc hardware-capability directories and `/usr/lib64` also fail.

This establishes that the problem is renderer sandbox filesystem access, not library discovery, missing symbols, or decoder registration.

### Why not allow openat?

The renderer seccomp policy delegates unspecified system calls to the baseline policy. The baseline policy denies `openat()`.

Allowing unrestricted `openat()` in the renderer would weaken Chromium's filesystem isolation. It is not an acceptable general fix.

The broker policy is also not a direct solution: the CUVID loader uses `dlopen()` and needs the dynamic loader to access the libraries.

### Proposed preloading solution

The renderer has a pre-sandbox initialisation point in:

```text
content/renderer/renderer_main_platform_delegate_linux.cc
```

`RendererMainPlatformDelegate::EnableSandbox()` calls:

```cpp
sandbox::policy::Sandbox::Initialize(
    sandbox::policy::SandboxTypeFromCommandLine(
        *base::CommandLine::ForCurrentProcess()),
    sandbox::policy::SandboxLinux::PreSandboxHook(), options);
```

The proposed change is to preload the NVIDIA libraries immediately before this call:

```cpp
#include <dlfcn.h>
```

```cpp
#if defined(__x86_64__)
  // Preload NVIDIA libraries before the renderer seccomp sandbox blocks
  // openat(). FFmpeg's CUVID loader will reuse these loaded libraries.
  dlopen("libcuda.so.1", RTLD_NOW | RTLD_LOCAL);
  dlopen("libnvcuvid.so.1", RTLD_NOW | RTLD_LOCAL);
#endif
```

This should allow the dynamic loader to reuse the already-loaded libraries when FFmpeg initialises CUVID after sandbox activation.

**Important:** This is a proposed change, not yet verified. The next test is to rebuild Chromium and run it normally, without `--no-sandbox`.

If the later CUVID `dlopen()` still attempts filesystem access, the next step is to investigate explicitly retaining and reusing the preloaded library handles. The renderer seccomp policy should not be weakened as a workaround.

## 7. Testing and diagnostics

### Test without sandbox

The following command has successfully launched Chromium and played UniFi Protect streams:

```bash
./out/HEVC/chrome \
  --user-data-dir=/tmp/chromium-cuvid-test \
  --enable-logging=stderr \
  --v=1 \
  --no-sandbox \
  2>&1 | tee /tmp/chromium-cuvid-test.log
```

Both H.264 and HEVC Protect streams were tested. NVIDIA decoder utilisation was observed increasing when switching to high-resolution streams.

The H.264 WebRTC path is worth distinguishing from this test: Chromium can use a separate software H.264 decoder for WebRTC. Hardware utilisation and the CUVID initialisation logs are more useful evidence of the FFmpeg CUVID path than simply observing that an H.264 stream plays.

### Useful log filters

```bash
grep -E 'CUVID DEBUG:|CUVID INIT:|CUVID TRACE:' \
  /tmp/chromium-cuvid-test.log
```

Check decoder selection and initialisation:

```bash
grep -E 'hevc_cuvid|h264_cuvid|avcodec_open2' \
  /tmp/chromium-cuvid-test.log
```

Check NVIDIA decoder utilisation:

```bash
nvidia-smi dmon
```

### Expected successful initialisation

The HEVC decoder previously reached:

```text
CUVID TRACE: before ff_decode_preinit codec=hevc_cuvid
CUVID TRACE: after ff_decode_preinit ret=0
CUVID TRACE: before ff_thread_init codec=hevc_cuvid
CUVID TRACE: after ff_thread_init ret=0
CUVID TRACE: entering codec init name=hevc_cuvid id=172
```

The earlier `ff_decode_preinit()` failure was caused by the missing HEVC bitstream filter registration and has been resolved.

## 8. Important implementation notes

- Keep the existing Chromium software decoding paths intact for codecs other than H.264 and HEVC.
- Do not replace the CUVID decoders with NVDEC hwaccel wrappers.
- Do not introduce a general renderer `openat()` seccomp exception.
- Do not revisit the VA-API path as a workaround. The NVIDIA VA-API route was investigated and did not provide the required working decode path.
- Keep the FFmpeg generated configuration copies synchronised with the generated build output.
- Preserve the distinction between the FFmpeg CUVID path and Chromium's separate WebRTC decoder paths.
- Keep temporary diagnostic logging until sandboxed playback is confirmed. Remove temporary debug instrumentation before producing the final patch, unless it is intentionally retained as useful diagnostics.

## 9. Final verification checklist

Before generating the final `.diff`:

- [ ] Build Chromium successfully with the renderer preloading change.
- [ ] Launch Chromium normally, without `--no-sandbox`.
- [ ] Confirm NVIDIA library preloading occurs before sandbox activation.
- [ ] Confirm CUVID initialises after sandbox activation.
- [ ] Confirm HEVC Protect streams play correctly.
- [ ] Confirm H.264 Protect streams play correctly.
- [ ] Confirm NVIDIA hardware decoder utilisation during playback.
- [ ] Confirm ordinary non-CUVID codecs still work.
- [ ] Confirm Chromium's renderer sandbox remains enabled.
- [ ] Remove temporary diagnostic instrumentation.
- [ ] Review all modified and generated files.
- [ ] Generate a consolidated patch with `git diff`.

## 10. Final patch generation

Once testing is complete, inspect the working tree:

```bash
git status --short
```

Review the changes:

```bash
git diff --stat
git diff
```

Generate the developer patch:

```bash
git diff > chromium-nvidia-cuvid.patch
```

The patch should include the source changes and any required generated FFmpeg configuration changes.

The README should be committed alongside the patch so that the build requirements, implementation decisions, and sandbox investigation are documented for future maintainers.
