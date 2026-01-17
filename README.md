# **FFmpeg HDR Metadata Extended Documentation**

This version adds support for **HDR [SIDE_DATA] insertion** via the **HEVC (H265) bitstream filter (bsf)**.

## **📖 Usage Guide (Windows PowerShell)**

Follow these three steps to extract video, inject HDR metadata, and remux the content.

### **Step 1: Extract Raw H265 Video Stream**

```powershell
.\ffmpeg -i .\HDR_test_01.mp4 -c:v copy -an .\HDR_test_01.h265
```

### **Step 2: Inject HDR10 Static Metadata (Mastering Display Metadata)**

Use the hevc\_metadata filter to insert color coordinates and luminance ranges:
```powershell
.\ffmpeg.exe -i .\HDR_test_01.h265 -c copy -tag:v hvc1 -movflags +faststart -bsf:v "hevc_metadata=colour_primaries=9:transfer_characteristics=16:matrix_coefficients=9:mastering_display=G(8500\,39849)B(6549\,2300)R(35400\,14599)WP(15635\,16449)L(10000000\,50)" final_fixed_native.hevc
```

### **Step 3: Remux to MP4 and Merge Audio**

```powershell
ffmpeg -r 60 -i .\final_fixed_native.hevc -i .\HDR_test_01.mp4 -map 0:v -map 1:a -c copy -map_metadata 1 -tag:v hvc1 -bsf:v hevc_metadata -brand mp42 -movflags +faststart final_fixed.mp4
```

---

**✅ Verification**

Use ffprobe to verify if the first frame contains the SIDE_DATA:

* **Before:** The frame=side_data_list output will be empty.  
```powershell
ffprobe -v quiet -select_streams v:0 -show_frames -read_intervals "%+#1" -show_entries frame=side_data_list .\HDR_test_01.mp4
[FRAME]
[/FRAME]
```

* **After:** The output should include side_data_type=Mastering display metadata along with the specific coordinates (e.g., red_x, green_x) and max_luminance values.
```powershell
ffprobe -v quiet -select_streams v:0 -show_frames -read_intervals "%+#1" -show_entries frame=side_data_list final_fixed.mp4
[FRAME]
[SIDE_DATA]
side_data_type=Mastering display metadata
red_x=35400/50000
red_y=14599/50000
green_x=8500/50000
green_y=39849/50000
blue_x=6549/50000
blue_y=2300/50000
white_point_x=15635/50000
white_point_y=16449/50000
min_luminance=50/10000
max_luminance=10000000/10000
[/SIDE_DATA]
[/FRAME]
```

---

**🛠️ Build Instructions for Windows (MSVC + MSYS2)**

To build this version from source, follow these steps:

### **1. Environment Setup**

1. Open Visual Studio Tools\VC\x64 Native Tools Command Prompt
2. Launch the **MSYS2 UCRT64** shell:
   - Edit msys2_shell.cmd enable(uncomment) set MSYS2_PATH_TYPE=inherit
   - C:\msys64\msys2_shell.cmd -ucrt64
3. Install requirements:
   - pacman -S nasm
   - pacman -S yasm
   - pacman -S diffutils

### **2. Configure**

```bash
./configure --prefix=/usr/local/ffmpegmsvc --target-os=win64 --arch=x86_64 --enable-shared --toolchain=msvc --enable-gpl --enable-debug
```

### **3. Edit config.mak (Critical Fixes)**
Before building, manually modify config.mak:

```
1. replace gsub(/\\/, "/") to gsub(/\\\\/, "/")
2. HOSTEXTRALIBS=-lm to HOSTEXTRALIBS=
```

```
3. 
SLIB_CREATE_DEF_CMD=LDFLAGS="$(LDFLAGS)" EXTERN_PREFIX="$(EXTERN_PREFIX)" $(SRC_PATH)/compat/windows/makedef $(SUBDIR)lib$(NAME).ver $(OBJS) > $$(@:$(SLIBSUF)=.def)

to

SLIB_CREATE_DEF_CMD=LDFLAGS="$(LDFLAGS)" EXTERN_PREFIX="$(EXTERN_PREFIX)" sh $(SRC_PATH)/compat/windows/makedef $(SUBDIR)lib$(NAME).ver @$$@.objs > $$(@:$(SLIBSUF)=.def)
```

### **4. Patch compat/windows/makedef (Critical Fixes)**
This script needs modifications to handle **response files** (fixing the "command line too long" error) and correctly invoke `lib.exe`.

**Modify `compat/windows/makedef`:**

1.  **Add Response File Support**:
    Inside the `for object in "$@"; do` loop (around line 38), add check for `@`:
    ```bash
    for object in "$@"; do
        if [ "${object#@}" != "${object}" ]; then
            continue
        fi
        # ... existing file check ...
    ```

2.  **Fix `lib.exe` Detection**:
    Replace the `AR` invocation logic (around line 54) to explicitly check for `lib.exe` and handle arguments correctly:
    ```bash
    use_lib_exe=0
    if [ -n "$AR" ]; then
        case "$AR" in
            *lib.exe|*lib)
                use_lib_exe=1
                ;;
        esac
    fi

    if [ -n "$AR" ] && [ $use_lib_exe -eq 0 ]; then
        $AR rcs ${libname} $@ >/dev/null
    else
        # AR is either unset or set to lib.exe
        # ... proper lib.exe invocation ...
        if [ -n "$AR" ]; then
            $AR ${machine_flag} -out:${libname} $@
        else
            lib.exe ${machine_flag} -out:${libname} $@
        fi
    fi
    ```

### **5. Patch ffbuild/library.mak (Fix Command Line Length)**
The `echo` command fails when the list of object files is too long (32k+ characters). Use GNU Make's `$(file ...)` function instead.

**Modify `ffbuild/library.mak`:**

Around line 79, replace the `echo` line with the `file` function:

```makefile
# Old line (fails):
# $(Q)echo $$(filter %.o,$$^) > $$@.objs

# New line (works):
$$(file >$$@.objs,$$(filter %.o,$$^))
```

### **6. Build and Install**

 1. Build
    - make -j$(nproc) > build_all.txt
    - if want to restart build run "make clean" or full clean run "make distclean".

2. Get Final Package
    - make install
    - The final binaries will be located in /usr/local/ffmpegmsvc.

---

FFmpeg README
=============

FFmpeg is a collection of libraries and tools to process multimedia content
such as audio, video, subtitles and related metadata.

## Libraries

* `libavcodec` provides implementation of a wider range of codecs.
* `libavformat` implements streaming protocols, container formats and basic I/O access.
* `libavutil` includes hashers, decompressors and miscellaneous utility functions.
* `libavfilter` provides means to alter decoded audio and video through a directed graph of connected filters.
* `libavdevice` provides an abstraction to access capture and playback devices.
* `libswresample` implements audio mixing and resampling routines.
* `libswscale` implements color conversion and scaling routines.

## Tools

* [ffmpeg](https://ffmpeg.org/ffmpeg.html) is a command line toolbox to
  manipulate, convert and stream multimedia content.
* [ffplay](https://ffmpeg.org/ffplay.html) is a minimalistic multimedia player.
* [ffprobe](https://ffmpeg.org/ffprobe.html) is a simple analysis tool to inspect
  multimedia content.
* Additional small tools such as `aviocat`, `ismindex` and `qt-faststart`.

## Documentation

The offline documentation is available in the **doc/** directory.

The online documentation is available in the main [website](https://ffmpeg.org)
and in the [wiki](https://trac.ffmpeg.org).

### Examples

Coding examples are available in the **doc/examples** directory.

## License

FFmpeg codebase is mainly LGPL-licensed with optional components licensed under
GPL. Please refer to the LICENSE file for detailed information.

## Contributing

Patches should be submitted to the ffmpeg-devel mailing list using
`git format-patch` or `git send-email`. Github pull requests should be
avoided because they are not part of our review process and will be ignored.
