# nodejs-mobile-prebuilt

Prebuilt `libnode.so` + headers, cross-compiled for Android from
[nodejs-mobile](https://github.com/nodejs-mobile/nodejs-mobile)'s own
`update22-9-0` branch (Node.js v22.9.0), published as downloadable GitHub
Release assets so any project can bundle a real, modern Node.js runtime in a
native Android app without setting up its own cross-compile toolchain.

nodejs-mobile's own official releases only go up to v18.20.4 -- the
`update22-9-0` branch is a genuine, real upstream effort to bring it to
Node.js v22, but it's an unfinished work-in-progress with no CI run of its
own on the official repo (its own `.github/workflows/build-mobile.yml`
only triggers on pushes/PRs into `main`, which this branch is neither) and
no published release. This repo builds it anyway, applies two small,
necessary patches (see below), and publishes the result.

## Why this exists

Recent YouTube extraction (via yt-dlp's own `--js-runtimes` mechanism, used
to solve signature/PO-token challenges) needs a real, modern JS runtime --
and yt-dlp itself has its own separate version floor
(`yt_dlp/utils/_jsruntime.py`'s `NodeJsRuntime.MIN_SUPPORTED_VERSION =
(22, 0, 0)`, a deliberate EOL-alignment policy bump, not a hard technical
requirement -- see
[yt-dlp/yt-dlp#16765](https://github.com/yt-dlp/yt-dlp/issues/16765)) that
nodejs-mobile's official v18.20.4 release can't satisfy at all. This repo
exists to build a real v22 so that floor is met honestly, instead of
patching yt-dlp's own version check down to accept an older runtime.

## The patches

### `patches/v8-handles-cwg2518-workaround.patch`

Touches exactly one file,
`deps/v8/src/handles/handles.h`. Upstream V8 source there has:

```cpp
#if defined(__clang__) && __clang_major__ >= 17
  static_assert(false, "This handle does not reference a heap object. ...");
#endif
```

relying on [CWG2518](https://reviews.llvm.org/D144285) (a C++ defect-report
fix making a literal `static_assert(false)` inside a discarded
`if constexpr` branch valid, rather than an unconditional hard error) and
gating on `__clang_major__ >= 17` as a proxy for "the compiler has that fix".

Android NDK r26's bundled Clang (17.0.2, an AOSP build "based on
r487747e") self-reports as major version 17 but does **not** actually have
that fix -- confirmed directly with a minimal, standalone reproduction of
the exact same construct against this same toolchain binary (completely
outside V8), which hard-errors identically. The patch replaces the literal
`false` with `sizeof(T) == 0` -- the standard, compiler-version-independent
portable idiom for this exact situation (syntactically dependent on the
template parameter, so it's only ever diagnosed once the branch is actually
taken for a real type, regardless of whether the compiler implements
CWG2518) -- and removes the now-unnecessary version guard entirely.

Every build dependency issue (Python version, `rsync`, `gcc-multilib`) is a
build *environment* gap, not something requiring a source change.

### `patches/android-build-parallel-cap.patch`

Touches `tools/android_build.sh`'s one `make -j $(getconf
_NPROCESSORS_ONLN)` line, capping it to `-j2`. Found on this repo's own
first real CI run: the `arm64-v8a` matrix leg was silently killed mid-build
-- no compiler error, no exit-code annotation anywhere in the job log --
while `x86_64` happened to succeed in the very same run. That's the
signature of the runner's own OOM killer, not a real source bug: a
standard GitHub-hosted runner's 4 detected cores (what `getconf
_NPROCESSORS_ONLN` reports) don't come with enough memory headroom for V8's
own famously memory-hungry per-translation-unit compiles at that
parallelism. `-j2` trades some wall-clock time for a much lower peak-memory
ceiling -- the same tradeoff this repo's first consumer, Vivace, already
made for an analogous from-source V8 build on a similarly memory-
constrained CI runner.

These two are the **only** source modifications anywhere in this pipeline.

## What gets built

| Target (script arg) | Package ABI name |
|----------------------|-------------------|
| `arm64`              | `arm64-v8a` (real devices) |
| `x86_64`             | `x86_64` (emulators) |

`armeabi-v7a` (32-bit) is skipped, matching this project's own sibling repo
[cpython-android-prebuilt](https://github.com/Sportacandy/cpython-android-prebuilt)'s
reasoning: real-device app submissions have been 64-bit-only for a while,
and this repo's first consumer only needs `arm64-v8a`.

Each release asset, `nodejs-mobile-v22.9.0-{abi}.tar.gz`, contains:

```
libnode.so                    # the built shared library
include/node/                 # node.h, node_version.h, v8-*.h, uv.h,
                               #   libplatform/, cppgc/, openssl/, ...
```

nodejs-mobile ships **no standalone `node` executable** at all -- it's an
embedding library, meant to be linked into your own native code and driven
via `node::Start(argc, argv)` (Node's own real `main()` entry point,
verified directly against upstream `src/node_main.cc`). Consumers wanting
a real, spawnable `node`-compatible binary need a thin wrapper of their
own (see Vivace's `src/android/vivace_node_main.cc` for a working
~10-line example) -- this repo does not build one, since the correct
wrapper shape (a shared library vs. a true executable, PIE flags, etc.)
is a consumer-side decision this repo shouldn't presume.

## Using a release in your own project

1. Download the `.tar.gz` for your target ABI from the
   [Releases](https://github.com/Sportacandy/nodejs-mobile-prebuilt/releases)
   page and extract it.
2. Link your own wrapper (or embedding code) against `libnode.so`; add
   `include/node/` to your include path.
3. Bundle `libnode.so` into your app's `nativeLibraryDir` (e.g. via Qt's
   `QT_ANDROID_EXTRA_LIBS`, or the Gradle-native equivalent) -- Android's
   noexec policy on app-private storage means this file must be part of
   the signed, installed package, not something extracted to
   `filesDir`/`cacheDir` at runtime.

## Building a new version

Trigger the **Build nodejs-mobile for Android** workflow manually (Actions
tab -> "Run workflow"), optionally giving it a different nodejs-mobile git
ref (default: the exact commit this repo's patch was verified against).
Pushing a `v*` tag on this repo also builds and publishes a release
automatically.

If nodejs-mobile's `update22-9-0` branch ever moves in a way that changes
`deps/v8/src/handles/handles.h`'s shape, `git apply` will fail loudly at
the patch step rather than silently applying wrong -- re-check the patch
against the new source before re-pinning `nodejs_mobile_ref`.

## License

The workflow/scripts/patch in this repo are MIT-licensed (see `LICENSE`).
The *contents* of the published tarballs are Node.js/V8 themselves (with
the one small patch above), under their own upstream licenses -- anyone
redistributing those tarballs (or apps that embed them) needs to comply
with those licenses, not this repo's MIT one.
