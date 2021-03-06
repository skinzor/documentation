---
layout:      docs
title:       Building
description: Instructions on building CopperheadOS from source and signing releases
priority:    1.0
---

# Building CopperheadOS from source
{: .no_toc}

* TOC
{:toc}

## Build dependencies

* x86\_64 Linux build environment (OS X is not supported, unlike AOSP)
* Android Open Source Project build dependencies
* Linux kernel build dependencies
* 16GiB of memory or more
* 200GiB of free storage space

## Supported devices

CopperheadOS currently has official support for the following devices:

* HiKey (hikey)
* HiKey 960 (hikey960)
* Nexus 5X (bullhead)
* Nexus 6P (angler)
* Pixel (sailfish)
* Pixel XL (marlin)
* Pixel 2 (walleye)
* Pixel 2 XL (taimen)

It can be ported to other Android devices with Treble support via the standard [device porting
process](/android/docs/building#device-porting-process). Most devices lack support for the
[security requirements](/android/docs/devices#minimum-requirements-for-copperheados-support)
needed to match how it works on the officially supported devices.

HiKey and HiKey 960 do not support verified boot and the current Android Open Source Project
implementation lacks support for updates. CopperheadOS targets them as development devices.

## Downloading source code

Since this is syncing the sources for the entire operating system and application layer, it will
use a lot of bandwidth and storage space.

You likely want to use the most recent stable tag, not the development branch, even for developing
a feature. It's easier to port between stable tags that are known to work properly than dealing
with a moving target.

### Development branch

The oreo-m2-s2-release branch is used for the HiKey, HiKey 960, Nexus 5X, Pixel, Pixel XL, Pixel
2, Pixel 2 XL and other devices:

    mkdir copperheados-oreo-m2-s2-release
    cd copperheados-oreo-m2-s2-release
    repo init -u https://github.com/CopperheadOS/platform_manifest.git -b oreo-m2-s2-release
    repo sync -j32

The oreo-m2-s2-release branch can be used for the Nexus 6P but it isn't yet officially supported
by this new development branch due to lack of a full Android Open Source Project release.

If your network is unreliable and `repo sync` fails, you can run the `repo
sync` command again as many times as needed for it to fully succeed.

### Stable release

Pick a specific build for a device from the [releases page](/android/releases) and download the
source tree. Note that some devices use different Android Open Source Project branches so they end
up with different tags. Make sure to use the correct tag for a device. For devices without
official support, use the latest tag for the Pixel 2 XL.

    mkdir copperheados-TAG_NAME
    cd copperheados-TAG_NAME
    repo init -u https://github.com/CopperheadOS/platform_manifest.git -b refs/tags/TAG_NAME

Verify the manifest:

    gpg --recv-keys 65EEFE022108E2B708CBFCF7F9E712E59AF5F22A
    gpg --recv-keys 4340D13570EF945E83810964E8AD3F819AB10E78
    cd .repo/manifests
    git verify-tag --raw $(git describe)
    cd ../..

Complete the source tree download:

    repo sync -j32

Verify the source tree:

    repo forall -c 'git verify-tag --raw $(git describe)' || echo Verification failed!

These instructions will be extended in the future to check the verify-tag output.

Note that the repo command itself takes care of updating itself and uses gpg to verify by default.

### Updating and switching branches/tags

To update the source tree, run the `repo init` command again to select the branch or tag and then
run `repo sync -j32` again. You may need to add `--force-sync` if a repository from switched from
one source to another, such as when CopperheadOS forks an additional Android Open Source Project
repository. You don't need to start over to switch between different branches or tags. You may
need to run `repo init` again to continue down the same branch since CopperheadOS only provides a
stable history via tags.

## Chromium and WebView

Before building CopperheadOS, you need to build Chromium for the WebView and *optionally* the
standalone browser app. CopperheadOS uses a hardened fork of Chromium for these. It needs to be
rebuilt when Chromium is updated or the CopperheadOS `chromium_patches` repository changes.

Chromium and the WebView are independent applications built from the Chromium source tree. The
CopperheadOS Chromium build is located at external/chromium and includes the WebView.

See [Chromium's Android build
instructions](https://www.chromium.org/developers/how-tos/android-build-instructions) for details
on obtaining the prerequisites.

    mkdir chromium
    cd chromium
    fetch --nohooks android --target_os_only=true

Sync to the latest stable release for Android:

    gclient sync --with_branch_heads -r 66.0.3359.158 --jobs 32

Apply the CopperheadOS patches on top of the tagged release:

    git clone https://github.com/CopperheadOS/chromium_patches.git
    cd src
    git am ../chromium_patches/*.patch

Note that we don't have our own public repository at the moment because Chromium is too large to
host it on GitHub or Bitbucket where we are hosting the other repositories.

Then, configure the build in the `src` directory:

    gn args out/Default

CopperheadOS configuration:

    target_os = "android"
    target_cpu = "arm64"
    is_debug = false

    is_official_build = true
    is_component_build = false
    symbol_level = 0

    ffmpeg_branding = "Chrome"
    proprietary_codecs = true

    android_channel = "stable"
    android_default_version_name = "66.0.3359.158"
    android_default_version_code = "335915852"

To build Monochrome, which provides both Chromium and the WebView:

    ninja -C out/Default/ monochrome_public_apk

The apk needs to be copied from `out/Default/apks/MonochromePublic.apk` into the Android source
tree at `external/chromium/prebuilt/arm64/MonochromePublic.apk`

Standalone builds of Chromium and the WebView can be done via the `chrome_modern_public_apk` and
`system_webview_apk` targets but those aren't used by CopperheadOS. The build system isn't set up
for including them and the standalone WebView isn't whitelisted in
`frameworks/base/core/res/res/xml/config_webview_packages`.

## Setting up the build environment

The build has to be done from bash as envsetup.sh is not compatible with other shells like zsh.

Set up the build environment:

    source script/copperhead.sh

Select the desired build target (`aosp_marlin` is the Pixel XL):

    choosecombo release aosp_marlin user

For a development build, you may want to replace `user` with `userdebug` in order to have better
debugging support. Production builds should be `user` builds as they are significantly more
secure and don't make additional performance sacrifices to improve debugging.

### Reproducible builds

To reproduce a past build, you need to export `BUILD_DATETIME` and `BUILD_NUMBER` to the values
set for the past build. These can be obtained from `out/build_date.txt` and `out/build_number.txt`
in a build output directory and the `ro.build.date.utc` and `ro.build.version.incremental`
properties which are also included in the over-the-air zip metadata rather than just the OS
itself.

The signing process for release builds is done after completing builds and replaces the dm-verity
trees, apk signatures, etc. and can only be reproduced with access to the same private keys. If
you want to compare to production builds signed with different keys you need to stick to comparing
everything other than the signatures.

## Extracting vendor files for Nexus and Pixel devices

***This section does not apply to HiKey or HiKey 960 as no extra vendor files are required.***

Extract the vendor files corresponding to the matching release:

    vendor/android-prepare-vendor/execute-all.sh -d DEVICE -b BUILD_ID -o vendor/android-prepare-vendor
    mkdir -p vendor/google_devices
    rm -rf vendor/google_devices/DEVICE
    mv vendor/android-prepare-vendor/DEVICE/BUILD_ID/vendor/google_devices/DEVICE vendor/google_devices

For the smaller Pixel (sailfish), there's both a marlin and sailfish directory to move:

    rm -rf vendor/google_devices/marlin
    mv vendor/android-prepare-vendor/sailfish/BUILD_ID/vendor/google_devices/marlin vendor/google_devices

Note that android-prepare-vendor is non-deterministic for apk and jar files where Google doesn't
provide them unoptimized / unstripped. This was unintentionally improved by Google for the Pixel
and Pixel XL since Google stopped including odex files in the main system image and they are now
provided as unstripped apk files.

The development branch currently requires modifying `build/core/build_id.mk` to the correct build
id for the vendor files, since it may be newer than the last version for the vendor files and the
generated vendor files have a very useful `BUILD_ID` version check to catch mistakes.

## Generating release signing keys

Keys need to be generated for resigning completed builds from the publicly available test keys.
The keys must then be reused for subsequent builds and cannot be changed without flashing the
generated factory images again which will perform a factory reset. Note that the keys are used for
a lot more than simply verifying updates and verified boot. Keys must be generated before building
for the Pixel and Pixel XL due to needing to provide the keys to the kernel build system, but this
step can be done after building for Nexus devices.

The keys should not be given passwords due to limitations in the upstream scripts. If you want to
secure them at rest, you should take a different approach where they can still be available to the
signing scripts as a directory of unencrypted keys. The sample certificate subject can be replaced
with your own information or simply left as-is.

The Nexus 5X, Nexus 6P, Pixel and Pixel XL use Android Verified Boot 1.0. The Pixel 2 and Pixel 2
XL use Android Verified Boot 2.0 (AVB). Follow the appropriate instructions below.

### Android Verified Boot 1.0

To generate keys for marlin (you should use unique keys per device variant):

    mkdir -p keys/marlin
    cd keys/marlin
    ../../development/tools/make_key releasekey '/C=CA/ST=Ontario/L=Toronto/O=CopperheadOS/OU=CopperheadOS/CN=CopperheadOS/emailAddress=copperheados@copperhead.co'
    ../../development/tools/make_key platform '/C=CA/ST=Ontario/L=Toronto/O=CopperheadOS/OU=CopperheadOS/CN=CopperheadOS/emailAddress=copperheados@copperhead.co'
    ../../development/tools/make_key shared '/C=CA/ST=Ontario/L=Toronto/O=CopperheadOS/OU=CopperheadOS/CN=CopperheadOS/emailAddress=copperheados@copperhead.co'
    ../../development/tools/make_key media '/C=CA/ST=Ontario/L=Toronto/O=CopperheadOS/OU=CopperheadOS/CN=CopperheadOS/emailAddress=copperheados@copperhead.co'
    ../../development/tools/make_key verity '/C=CA/ST=Ontario/L=Toronto/O=CopperheadOS/OU=CopperheadOS/CN=CopperheadOS/emailAddress=copperheados@copperhead.co'
    cd ../..

Generate the verity public key:

    make -j20 generate_verity_key
    out/host/linux-x86/bin/generate_verity_key -convert keys/marlin/verity.x509.pem keys/marlin/verity_key

Generate verity keys in the format used by the kernel for the Pixel and Pixel XL:

    openssl x509 -outform der -in keys/marlin/verity.x509.pem -out kernel/google/marlin/verity_user.der.x509

The same kernel and device repository is used for the Pixel and Pixel XL. There's no separate
sailfish kernel.

### Android Verified Boot 2.0 (AVB)

To generate keys for taimen (you should use unique keys per device variant):

    mkdir -p keys/taimen
    cd keys/taimen
    ../../development/tools/make_key releasekey '/C=CA/ST=Ontario/L=Toronto/O=CopperheadOS/OU=CopperheadOS/CN=CopperheadOS/emailAddress=copperheados@copperhead.co'
    ../../development/tools/make_key platform '/C=CA/ST=Ontario/L=Toronto/O=CopperheadOS/OU=CopperheadOS/CN=CopperheadOS/emailAddress=copperheados@copperhead.co'
    ../../development/tools/make_key shared '/C=CA/ST=Ontario/L=Toronto/O=CopperheadOS/OU=CopperheadOS/CN=CopperheadOS/emailAddress=copperheados@copperhead.co'
    ../../development/tools/make_key media '/C=CA/ST=Ontario/L=Toronto/O=CopperheadOS/OU=CopperheadOS/CN=CopperheadOS/emailAddress=copperheados@copperhead.co'
    openssl genrsa -out avb.pem 2048
    ../../external/avb/avbtool extract_public_key --key avb.pem --output avb_pkmd.bin
    cd ../..

The `avb_pkmd.bin` file isn't needed for generating a signed release but rather to set the public
key used by the device to enforce verified boot.

## Building

Incremental builds (i.e. starting from the old build) usually work for development and are the
normal way to develop changes. However, there are cases where changes are not properly picked up
by the build system. For production builds, you should remove the remnants of any past builds
before starting, particularly if there were non-trivial changes:

    rm -r out

Start the build process, with -j# used to set the number of parallel jobs to the number of CPU
threads. You also need 2-4GiB of memory per job, so reduce it based on available memory if
necessary:

    make target-files-package -j20

### Faster builds for development use only

The normal production build process involves building a target files package to be resigned with
secure release keys and then converted into factory images and/or an update zip via the sections
below. If you have a dedicated development device with no security requirements, you can save time
by using the default make target, leaving the bootloader unlocked and flashing the raw images that
are signed with the default public test keys:

    make -j20

Technically, you could generate test key signed update packages. However, there's no point of
sideloading update packages when the bootloader is unlocked and there's no value in a locked
bootloader without signing the build using release keys, since verified boot will be meaningless
and the keys used to verify sideloaded updates are also public. The only reason to use update
packages or a locked bootloader without signing the build with release keys would be testing that
functionality and it makes a lot more sense to test it with proper signing keys rather than the
default public test keys.

## Generating signed factory images and full update packages

For the Pixel and Pixel XL, build the tool needed to generate A/B updates:

    make -j20 brillo_update_payload

For HiKey and HiKey 960, build dumpkey:

    make -j20 dumpkey

Generate a signed release build with the release.sh script:

    script/release.sh marlin

The factory images and update package will be in `out/release-marlin-$BUILD_NUMBER`. The update
zip performs a full OS installation so it can be used to update from any previous version. More
efficient incremental updates are used for official over-the-air CopperheadOS updates and can be
generated by keeping around past signed `target_files` zips and generating incremental updates
from those to the most recent signed `target_files` zip.

## Prebuilt code

Like the Android Open Source Project, CopperheadOS contains some code that's built separately and
then bundled into the source tree as binaries. Ideally, everything would be built-in tree with the
AOSP build system but it's not always practical.

### Kernel

Unlike AOSP, CopperheadOS builds the kernel as part of the operating system rather than bundling a
pre-built kernel image.

### Clang toolchain

Clang is not built via the standard manifest. It can be built for the oreo-m2-s2-release branch by
using `oreo-dr1-release-llvm` (*not* `oreo-m2-s2-release-llvm`) as the `repo init` target:

    mkdir oreo-dr1-release-llvm
    cd oreo-dr1-release-llvm
    repo init -u https://github.com/CopperheadOS/platform_manifest.git -b oreo-dr1-release-llvm
    repo sync -j32
    source script/copperhead.sh

The main component is clang itself, but other LLVM components can also be rebuilt if relevant
changes are made:

    make -j20 clang

The existing build can be reused other than components that are being replaced. The output can be
copied over top of the existing clang-copperhead toolchain version.

    cp out/host/linux-x86/bin/clang ../oreo-m2-s2-release/prebuilts/clang/host/linux-x86/clang-copperhead/bin/clang.real

Alternatively, a full build can be made and the build system updated to point at the new
directory. Documentation on a full rebuild may be added here later but it's not part of the
regular CopperheadOS build process since there are currently only changes to Clang.

### F-Droid

F-Droid is built from the [CopperheadOS fdroidclient
repository](https://github.com/CopperheadOS/fdroidclient) and then bundled as an apk in the
external/F-Droid repository.

The privileged extension built from source from the packages/apps/F-Droid/privileged-extension
repository as part of the normal build process.

### Other apps

Etar, Offline Calendar, Net Monitor and Silence are simply built from the latest upstream tag and
bundled as apks into external/ repositories. There are no modifications to these apps.

## Device porting process

Early AOSP port:

- implement userdebug test-keys builds, likely with android-prepare-vendor
- implement full signing for proper release-keys builds (including verified boot)
- configure and enable verified boot on the device

AOSP testing:

- test verified boot for all OS partitions (vbmeta, boot, dtbo, system, vendor for the Pixel 2)
- test verified boot rollback protection
- test key attestation support for verified boot attestation
- test that over-the-air updates work properly including updating all firmware images and vendor.img (with verified boot enabled to catch issues)
- test that incremental over-the-air updates work properly
- run CTS and record results

AOSP polish:

- improve port as needed until it's solid enough to work on CopperheadOS support

Early CopperheadOS port:

- minimal changes in the device repository if necessary to get it working with CopperheadOS changes
- work around or fix any latent device-specific memory corruption bugs that are occurring in
  regular use and break with CopperheadOS exploit mitigations

Early CopperheadOS testing:

- run through the AOSP testing step again with CopperheadOS
- compare CTS results with AOSP and make sure the difference matches the expected list of failures from intentional breaks in compatibility and AOSP bugs uncovered by mitigations that are not yet fixed

Full CopperheadOS port:

- port all device-specific changes to the device repository (Pixel 2 as the reference)
- integrate device into release, repository management and over-the-air update server scripting
- adapt device-specific extensions to SELinux policy to CopperheadOS SELinux policy hardening as necessary
- implement custom kernel builds
- Implement stable-base branch with all kernel.org LTS kernel changes cherry-picked + conflicts and other issues resolved. Keep empty commits, mark each commit with conflicts resolved at the bottom of the commit message with \[CopperheadOS\] and uncomment the automatically generated Conflicts list below that. See [the Pixel 2 kernel stable-base branch](https://github.com/CopperheadOS/kernel_google_wahoo/tree/oreo-m2-s2-release-stable-base) for an example. This branch should be kept rebased with history preserved via tags for each release.
- strip down kernel configuration to a minimum (Pixel 2 defconfig as the reference)
- implement monolithic kernel builds with modules disabled
- implement Clang-compiled kernel builds using the CopperheadOS Clang toolchain
- enable -fsanitize=local-init compiler feature for the kernel
- apply other CopperheadOS kernel hardening changes on top of the kernel.org LTS cherry-picks. Use the [Pixel 2 kernel branch](https://github.com/CopperheadOS/kernel_google_wahoo/tree/oreo-m2-s2-release) as a reference. This branch should be kept rebased with history preserved via tags for each release.
- implement specialized scanning MAC randomization for the Wi-Fi driver if not already implemented
- implement specialized associated MAC randomization for the Wi-Fi driver if not already implemented (Pixel as the reference for qcacld-2.0, Pixel 2 as the reference for qcacld-3.0)

Full CopperheadOS testing:

- repeat the early CopperheadOS testing step again now that device-specific changes are ported

## Redistribution

CopperheadOS kernel code is GPL2, so derivatives using only the kernel changes simply need to
respect the usual GPL2 rules by making the sources needed to build the kernel available, etc.

CopperheadOS userspace code is primarily licensed under the [Creative Commons
Attribution-NonCommercial-ShareAlike 4.0
International](https://creativecommons.org/licenses/by-nc-sa/4.0/) license so a commercial license
is required to earn money from derivatives using the userspace code. Licensing can be based on
revenue sharing so don't be afraid to contact sales@copperhead.co for small scale commercial
licensing.

CopperheadOS art and branding is primarily licensed under the [Creative Commons
Attribution-NonCommercial-NoDerivatives 4.0
International](https://creativecommons.org/licenses/by-nc-nd/4.0/) license. Derivatives that are
being distributed need to replace the boot animation, wallpaper and other art / branding. The
current branding to replace, which will expand over time:

* CopperheadOS boot animation (remove to use AOSP boot animation, or replace it)
* CopperheadOS wallpapers (remove to use AOSP default wallpapers)
* CopperheadOS Chromium branding (revert to Chromium branding, or replace it)

The 'CopperheadOS' branding itself is trademarked. Derivatives should come up with their own
project name and globally replace 'CopperheadOS' with 'NewProjectName' if they're being
distributed. They should state that they use code based on CopperheadOS / ported from CopperheadOS
but shouldn't claim to actually be CopperheadOS itself. The current list of strings to replace,
which will expand over time:

* ro.build.user and ro.build.host system properties are set to copperheados for reproducible
  builds and should be set to a different value
* kernel build user and host are set to copperheados for reproducible builds and should be set to
  a different value
* Clang toolchain version string (Android was replaced with CopperheadOS)
* keyboard (packages/inputmethods/LatinIME) branding strings (Android Open Source Project was
  replaced with CopperheadOS)
* update client (packages/apps/Updater) branding strings
* recovery branding string (Android was replaced with CopperheadOS)
