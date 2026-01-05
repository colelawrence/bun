# Build Release Workflow - Learnings & Troubleshooting

This document captures learnings from setting up GitHub Actions release builds for Bun.

## Key Differences: GitHub Actions vs Bun's Buildkite CI

Bun's official Buildkite CI uses a **three-stage build process**:
1. `BUN_CPP_ONLY=ON` - Builds C++ parts only
2. `build-zig` - Builds Zig parts (cross-compiled on Linux ARM64 Alpine)
3. `BUN_LINK_ONLY=ON` - Links everything together

Our GitHub Actions workflow does a **full single-stage build** (`bun run build:release`), which takes different code paths in CMake and can expose issues not seen in Buildkite.

---

## Issues Encountered & Solutions

### 1. LLVM 19 Required on macOS

**Error:** `CMAKE_C_COMPILER=CMAKE_C_COMPILER-NOTFOUND`

**Cause:** Bun's CMake (`cmake/tools/SetupLLVM.cmake`) explicitly looks for LLVM 19 at `/opt/homebrew/opt/llvm@19/bin` for clang, clang++, llvm-ar, dsymutil, etc.

**Solution:** Must install `llvm@19` via Homebrew:
```yaml
brew install llvm@19
```

---

### 2. JSC Template Undefined Variable Warning (llvm-19)

**Error:**
```
error: instantiation of variable 'JSC::JSGenericTypedArrayView<JSC::Uint8Adaptor>::s_info' 
required here, but no definition is available [-Werror,-Wundefined-var-template]
```

**Cause:** JavaScriptCore's `JSGenericTypedArrayView.h` forward-declares `s_info` with a comment saying "This is never accessed directly". LLVM 19's stricter checking flags this as an error when `-Werror` is enabled.

**Solution:** Add `-Wno-error=undefined-var-template` to `CXXFLAGS`:
```yaml
CXXFLAGS: "-Wno-error=undefined-var-template"
```

**Alternative (code fix):** Change `&JSC::JSUint8Array::s_info` to `JSC::JSUint8Array::info()` in source files.

---

### 3. C++23 Ranges Header Issue with `#define private public`

**Error:**
```
__ranges/lazy_split_view.h:143:10: error: '__outer_iterator' redeclared with 'public' access
```

**Cause:** Bun's `src/bun.js/bindings/v8/real_v8.h` uses `#define private public` to access V8 internals. This breaks standard library headers that have `private` forward declarations. The file pre-includes many stdlib headers before the macro, but `<ranges>` was missing.

**Solution:** Add `#include <ranges>` to the pre-include list in `real_v8.h`:
```cpp
// In src/bun.js/bindings/v8/real_v8.h, before #define private public:
#include <ranges>
```

---

### 4. CMake register_command Missing TARGETS

**Error:**
```
CMake Error at cmake/Globals.cmake:454 (message):
  register_command: TARGETS or SOURCES is required
Call Stack: cmake/targets/BuildBun.cmake:1458
```

**Cause:** The `register_command` function in `cmake/Globals.cmake` requires either `TARGETS` or `SOURCES` to be specified for dependency tracking. Some `register_command` calls with `TARGET_PHASE POST_BUILD` were missing this.

**Solution:** Add `TARGETS ${bun}` to the problematic `register_command` calls:
```cmake
register_command(
  TARGET
    ${bun}
  TARGETS          # <-- Added this
    ${bun}
  TARGET_PHASE
    POST_BUILD
  ...
)
```

---

### 5. Linux Dependencies

**Required packages for Ubuntu:**
- GCC 13+ (for C++23 and libstdc++)
- LLVM 19 (via `llvm.sh` script)
- libatomic1, libc6-dev

```yaml
sudo add-apt-repository ppa:ubuntu-toolchain-r/test -y
sudo apt-get install -y \
  gcc-13 g++-13 libgcc-13-dev libstdc++-13-dev \
  libatomic1 libc6-dev
```

---

### 6. macOS Runner Availability

- `macos-14` - ARM64 (Apple Silicon) - Free tier
- `macos-14-large` - x64 (Intel) - **Paid runner** (commented out)
- `macos-13` - **Retired** as of late 2024

---

### 7. Linux Disk Space Issues

**Error:**
```
System.IO.IOException: No space left on device
```

**Cause:** GitHub-hosted Ubuntu runners have ~14GB of free disk space, but Bun's build requires ~30GB due to:
- Large WebKit/JSC dependencies
- Debug symbols and intermediate object files
- LLVM 19 installation (~2GB)

**Solution:** Add a disk cleanup step at the start of Linux builds:
```yaml
- name: Free disk space (Linux)
  if: matrix.platform == 'linux'
  run: |
    # Remove large pre-installed software we don't need
    sudo rm -rf /usr/share/dotnet          # ~6GB
    sudo rm -rf /usr/local/lib/android     # ~10GB
    sudo rm -rf /opt/ghc                   # ~2GB
    sudo rm -rf /opt/hostedtoolcache/CodeQL
    sudo rm -rf /usr/local/share/boost
    sudo rm -rf /usr/share/swift
    sudo apt-get clean
```

This frees up ~20GB of additional space.

---

## Final Working CXXFLAGS

```yaml
# macOS
CXXFLAGS: "-Wno-error=undefined-var-template --stdlib=libc++"

# Linux
CXXFLAGS: "-Wno-error=undefined-var-template"
```

---

## Build Times (Approximate)

- macOS ARM64: ~20-30 minutes
- Linux x64: ~15-25 minutes  
- Linux ARM64: ~20-30 minutes

---

## Useful References

- Bun's Buildkite CI: `.buildkite/ci.mjs`
- LLVM setup: `cmake/tools/SetupLLVM.cmake`
- Build options: `cmake/Options.cmake`
- Bootstrap script: `scripts/bootstrap.sh`
- CONTRIBUTING.md - Official build instructions
