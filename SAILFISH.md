# SailfishOS patch set (`whisperfork`)

This branch carries the patch set used to build the static `libwebrtc.a`
artifacts that [Whisperfish](https://whisperfish.org) links into its
SailfishOS builds. It is based on `main` at tag `7871n` (commit
`25e48ee7`), the revision pinned by
[RingRTC](https://github.com/signalapp/ringrtc) v2.72.1.

The artifacts are published at https://nas.rubdos.be/~rsmet/webrtc/ as
`libwebrtc-<arch>-<sha384>.a`; the manifest below lets you verify that
they are exactly `this branch + patches/`, nothing else.

Signal builds for its own lld/compiler-rt toolchain; these artifacts must
be consumable by the older SailfishOS cross-binutils. That is the theme
of the whole patch set.

## What is patched where

- **In this branch (webrtc repo):** `BUILD.gn` — suppress OpenSSL 3
  deprecation warnings (`-Wno-deprecated-declarations`) in the
  `external_ssl_library` config. Building against system OpenSSL 3.x
  (`rtc_build_ssl=false`) otherwise trips `-Werror` on APIs that OpenSSL
  3 marks deprecated (`RSA_new`, `EC_KEY_*`, ...). This belongs upstream;
  candidate for a PR to signalapp/webrtc.
- **`patches/0001` — chromium build config (`src/build` dep):** disable
  ELF CREL. Upstream compiles with `-Wa,--crel` on all Linux targets
  except arm/s390x; CREL relocations require binutils ≥ 2.44 (or lld),
  and the SailfishOS binutils is older and refuses the objects
  ("skipping incompatible"). Drop when the SDK ships binutils ≥ 2.44.
- **`patches/0002` — libyuv (`src/third_party/libyuv` dep):** disable
  SME. libyuv builds SME kernels for all arm64 Linux targets
  (`-march=armv9-a+i8mm+sme`); their runtime helpers
  (`__arm_tpidr2_save` & co.) only exist in compiler-rt or libgcc ≥ 14,
  the SailfishOS libgcc lacks them, and no SailfishOS device has SME
  hardware anyway. Drop when the SDK ships GCC ≥ 14 (verify on device).

The two patch files cannot be committed here: they modify files in the
gclient-fetched dependency checkouts (`src/build` and
`src/third_party/libyuv`), which are separate git repositories. Apply
them after `gclient sync` from the `src/` root with `patch -p1`.

## Reproducing the artifacts

```sh
# 1. gclient workspace (or reuse ringrtc's bin/env.sh flow)
cat > .gclient <<EOF
solutions = [
  {
    "name": "src",
    "url": "https://github.com/whisperfish/webrtc",
    "deps_file": "DEPS",
    "managed": False,
    "custom_deps": {},
  },
]
EOF
export PATH=$HOME/src/depot_tools:$PATH
gclient sync --no-history --jobs 16 --with_tags --revision src@7871n
# DEPS are identical to this branch's base; then:
cd src && git checkout whisperfork   # in-tree patch above
cd .. && gclient runhooks --jobs 4

# 2. RingRTC's rffi headers must be present (ringrtc checkout matching
# the webrtc revision, i.e. v2.72.1):
ln -sfn $HOME/src/ringrtc/src/rust/src/webrtc_include src/ringrtc/rffi/api

# 3. Apply the dep-repo patches:
(cd src && for p in patches/*.patch; do patch -p1 < $p; done)

# 4. OpenSSL source trees for the include path: the tarball's include/
# is used as-is, except OpenSSL 3.6.5, which needs a one-time
# ./Configure && make build_generated to generate opensslv.h.

# 5. Per arch (arm arm64 x86 x64) and ssl variant (111 322 365):
gn gen "out/<ssl>-<arch>" --args='
    is_debug=false
    target_cpu="<arch>"
    rtc_build_examples=false rtc_build_tools=false
    rtc_use_x11=false rtc_enable_sctp=false
    rtc_libvpx_build_vp9=true
    rtc_disable_trace_events=true
    use_siso=false rtc_opus_support_dred=true
    rtc_include_tests=false rtc_enable_protobuf=false
    rtc_disable_metrics=true symbol_level=1
    rtc_build_ssl=false libsrtp_build_boringssl=false
    rtc_ssl_root="<openssl include>"
    libsrtp_ssl_root="<openssl include>"
'
ninja -C "out/<ssl>-<arch>"
# -> out/<ssl>-<arch>/obj/libwebrtc.a
```

The gn args mirror Signal's own release builds (`bin/build-desktop` in
ringrtc), except `use_siso=false` (plain ninja) and the SSL overrides.

## Manifest

sha384 of `out/<ssl>-<arch>/obj/libwebrtc.a` for this branch plus
`patches/`, against OpenSSL 1.1.1w (`111`), 3.2.2 (`322`) and 3.6.5
(`365`):

| ssl | arm | arm64 | x86 | x64 |
|-----|-----|------|-----|-----|
| 111 | 3ee0e141db54729e3224f95032f02672f277fe462536d734bf4c21c42e4cad373e773a7ee327222bd1aba2b907bf9443 | a30ec6e40c37fb61ec608373d68b2ce4e8c82cbdacc05967e37a293fe8f9bf16266d88b9b408ca409027d806bcb7d269 | fbb16c2be5db422da03b726a8aefe4da99b1eecc5c27ebada2f49bc7c83144549883721b745a1738886ff2ce2010ca7c | ee8887794b81acabe0025376de561bd32bb6736361711b074dd4f38889f4f835b8f1250faf05870ce58c682b2b55f1de |
| 322 | 75ef9d0b72909384a1d61203189e757df80c977a4247f0b3eb10ea4fc97dee144b5b912e7c2493b0c158cc28fee8aa39 | b470494b7351daf9ba7ef71414ff9784d0f5f12d12bf0a5efaa408b5189147dc38444fcf26a669ca5639a8b46add179d | 04660ec1b8f9ed13ee2dd7009a42b1f03af2841b56a32d058ac42e2afa5b3c1d2f5a8de5674771740273d2c0772b4994 | fd7a3d2d1c056ddac3e9ba4c4ef3691c8bc7db09e3c50ccbedc7ba92180394287f7ecf82d31a47fc25c9c5fa59171b98 |
| 365 | 7a86dacfc4dea3fd94284ee22c6221ed9c4858ae23cd05bea8c335f25d0f3881ecf92fceacee346b2201e2205af79c3f | 0de6c8cec509cbaa7f5a33760ae87fdfac0e8e80c63247572064da11cf24ee5bdccc258f29486d016b2883fd8f2152c1 | 8768578decb49f0cc72f7f43ed6fcf1ec9174c4b17aa5b24b8ab2d06866ed3574ccaa95a99641521285c13311b463b12 | c138c158a4ddb340fec0795fabb4013c5fb2a90611f3de79601f36e26a2af1ac5c50d1ce4351ab5dd1f7f061b7bf4ab5 |

The `<ssl>` variants differ only in the OpenSSL headers used for
compilation. The `111` variant links against the SailfishOS system
`libssl.so.1.1`; the `322`/`365` variants are matched to the OpenSSL
statically bundled in the Whisperfish package (openssl-sys / sqlcipher).
