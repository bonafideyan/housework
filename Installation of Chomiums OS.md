# Installation of Chomiums OS

[Home page](https://www.chromium.org/chromium-os)

[Guide](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md)

## Preparation

- Network: you need access google.com to get sources.
- Storage: more than 50G free disk.

Because my box has poor resouces, and I have none of a cloud offerings on which I
could configure a ss server too, I seeked help and got one with 200Mbps bandwith
 and 500G disk.

I compiled shadowsocksr-libev. But the machine can access google.com, so I have
not tried to get a cloud server to deploy it.


## Repository

According the guide, I installed the requested packages and configured them.
```
sudo apt-get install git-core gitk git-gui curl lvm2 thin-provisioning-tools \
     python-pkg-resources python-virtualenv python-oauth2client
```
```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=$PATH:/path/to/depot_tools
```
```
git config --global user.name "John Doe"
git config --global user.email "jdoe@email.com"
git config --global core.autocrlf false
git config --global core.filemode false
git config --global color.ui true
```
```
cd /tmp
cat > ./sudo_editor <<EOF
#!/bin/sh
echo Defaults \!tty_tickets > \$1          # Entering your password in one shell affects all shells 
echo Defaults timestamp_timeout=180 >> \$1 # Time between re-requesting your password, in minutes
EOF
chmod +x ./sudo_editor 
sudo EDITOR=./sudo_editor visudo -f /etc/sudoers.d/relax_requirements
```
And verify default permission.
```
$umask
0022
```
It's time to download repo.
```
mkdir -p ${HOME}/chromiumos
repo init -u https://chromium.googlesource.com/chromiumos/manifest.git --repo-url https://chromium.googlesource.com/external/repo.git -g minilayout
repo sync -j4
```

## Compilation

### First Attempt
```
cros_sdk
```
In this stage, I got many errors,
```
 fatal: unable to access 'https://chromium.googlesource.com/chromiumos/third_party/cups.git/': Couldn't resolve host 'chromium.googlesource.com'
 ```
 I check again and again, the DNS is not steady, so I try restart it,
```
sudo systemctl status systemd-resolved
systemd-resolve --status
    ...
    DNSSEC setting: yes
    DNSSEC supported: no
    ...
sudo systemctl restart systemd-resolved
```
But that did not resolve the elementary problem, At last, I realized that maybe
DNSSEC is culprit, so I modified /etc/systemd/resolved.conf and restart DNS service.
```
DNSSEC=no
```
It seems that connection got a mitigation. 

```
flashmap-0.3-r24: make -j32 
flashmap-0.3-r24: make: *** No targets specified and no makefile found.  Stop.
flashmap-0.3-r24:  * ERROR: sys-apps/flashmap-0.3-r24::chromiumos failed (compile phase):
flashmap-0.3-r24:  *   emake failed
```
```
update_payload-0.0.1-r143: OSError: [Errno 2] No such file or directory: 'scripts/update_payload/*.py'
update_payload-0.0.1-r143:  * ERROR: chromeos-base/update_payload-0.0.1-r143::chromiumos failed (install phase):
update_payload-0.0.1-r143:  *   doins failed
```
```
vboot_reference-1.0-r1589: make -j32 SRCDIR=/var/tmp/portage/chromeos-base/vboot_reference-1.0-r1589/work/vboot_reference-1.0 LIBDIR=lib64 ARCH=amd64 TPM2_MODE= PD_SYNC= USE_MTD= MINIMAL= NO_BUILD_TOOLS= DEV_DEBUG_FORCE= BUILD=/var/tmp/portage/chromeos-base/vboot_reference-1.0-r1589/work/build-main all 
vboot_reference-1.0-r1589: make: *** No rule to make target 'all'.  Stop.
vboot_reference-1.0-r1589:  * ERROR: chromeos-base/vboot_reference-1.0-r1589::chromiumos failed (compile phase):
vboot_reference-1.0-r1589:  *   emake failed
```
Because I'm new to portage build, I won't fix them now then decided use a release
branch.
```
cros_sdk --delete
rm -rf *
rm -rf .*
```

### Second Attempt
```
repo init \
  -u https://chromium.googlesource.com/chromiumos/manifest.git \
  -b release-R73-11647.B \
  --repo-url https://chromium.googlesource.com/external/repo.git \
  -g minilayout
repo sync -j10
```
This time I entered chroot sucesslly. then keen to build packages,
```
cros_sdk
~/trunk/src/scripts $ export BOARD=x86-generic
~/trunk/src/scripts $ ./setup_board --board=${BOARD}
```
Here It said git repo corrupt,
```
 * Existing git repo corrupt, removing and initialing clean checkout in /var/cache/chromeos-cache/distfiles/target/egit-src/external/libcxx
 * ERROR: sys-libs/libcxx-7.0.0-r5::chromiumos failed (unpack phase):
 *   git-2_initial_clone: can't fetch from https://android.googlesource.com/external/libcxx.git
```
I checked and modified the url,
```
diff --git a/sys-libs/libcxx/libcxx-7.0.0.ebuild b/sys-libs/libcxx/libcxx-7.0.0.ebuild
index 6bcedc941dd..4423e834928 100644
--- a/sys-libs/libcxx/libcxx-7.0.0.ebuild
+++ b/sys-libs/libcxx/libcxx-7.0.0.ebuild
@@ -11,7 +11,7 @@ PYTHON_COMPAT=( python2_7 )
 inherit cros-constants
 
 CROS_WORKON_REPO=${CROS_GIT_AOSP_URL}
-CROS_WORKON_PROJECT="external/libcxx"
+CROS_WORKON_PROJECT="platform/external/libcxx"
 CROS_WORKON_LOCALNAME="../aosp/external/libcxx"
 CROS_WORKON_BLACKLIST="1"
```
Then the last step can passed. (Maybe I could set an environment,
CROS_GIT_AOSP_URL=..., to avoid modify sources, but at that time I have not thought
of it.), but I encounted dependency problem,
```

!!! CONFIG_PROTECT is empty for '/build/x86-generic/'

These are the packages that would be merged, in order:

Calculating dependencies  ... done!

!!! The ebuild selected to satisfy "chromeos-base/chromeos-chrome" for /build/x86-generic/ has unmet requirements.
- chromeos-base/chromeos-chrome-73.0.3683.59_rc-r1::chromiumos USE="accessibility autotest build_tests buildcheck cfi chrome_debug chrome_remoting cups debug_fission evdev_gestures fonts highdpi libcxx lld nacl opengles reorder_text_sections runhooks smbprovider thinlto v4l2_codec vaapi xkbcommon -afdo_use -app_shell -asan (-authpolicy) -chrome_debug_tests -chrome_internal -chrome_media -clang -clang_tidy -component_build -gold -goma -hardfp -internal_gles_conform -jumbo -mojo (-neon) -new_tcmalloc -oobe_config -opengl -v4lplugin -verbose -vtable_verify" OZONE_PLATFORM="default_gbm gbm -caca -cast -default_caca -default_cast -default_egltest -default_test -egltest -test" OZONE_PLATFORM_DEFAULT="gbm -caca -cast -egltest -test"

  The following REQUIRED_USE flag constraints are unsatisfied:
    libcxx? ( clang ) thinlto? ( clang )

  The above constraints are a subset of the following complete expression:
    asan? ( clang ) at-most-one-of ( gold lld ) cfi? ( thinlto ) clang_tidy? ( clang ) libcxx? ( clang ) thinlto? ( clang any-of ( gold lld ) ) afdo_use? ( clang ) exactly-one-of ( ozone_platform_default_gbm ozone_platform_default_cast ozone_platform_default_test ozone_platform_default_egltest ozone_platform_default_caca )

(dependency required by "chromeos-base/autotest-chrome-0.0.1-r7406::chromiumos" [ebuild])
(dependency required by "chromeos-base/autotest-all-0.0.1-r48::chromiumos[-chromeless_tests,-chromeless_tty]" [ebuild])
(dependency required by "chromeos-base/autotest-all" [argument])
ERROR   : emerge detected broken ebuilds. See error message above.
```
I had no clue and asked to irc, got that clang is needed, but Calculating dependencies
said no, I decided try another release.

Release-R60-9592.B with minilayout spitted out more errors.
and release-R73-11647.B without minilayout used the partition up.
I reverted to release-R73-11647.B with minilayout and linked /opt/chroot to
~/chromiumos/chroot. But the problem keep there, after searching of answer, I
got that x86-generic board is not maintained well. so I tried amd64-generic board.

This time I went down to build packages, happily.
```
cros_sdk --delete
cros_sdk
~/trunk/src/scripts $ export BOARD=amd64-generic
~/trunk/src/scripts $ ./setup_board --board=${BOARD}
~/trunk/src/scripts $ ./set_shared_user_password.sh
~/trunk/src/scripts $ ./build_packages --board=${BOARD}
```
But still had errors that I can't fix. except an invalid URL.
```

diff --git a/chromeos-base/libmojo/libmojo-462023.ebuild b/chromeos-base/libmojo/libmojo-462023.ebuild
index 49b74acb2b4..c2882d21eb2 100644
--- a/chromeos-base/libmojo/libmojo-462023.ebuild
+++ b/chromeos-base/libmojo/libmojo-462023.ebuild
@@ -3,7 +3,8 @@
 
 EAPI="5"
 
-CROS_WORKON_PROJECT="aosp/platform/external/libmojo"
+CROS_WORKON_REPO="https://android.googlesource.com"
+CROS_WORKON_PROJECT="platform/external/libmojo"
 CROS_WORKON_COMMIT="02d8c056c1f7396f54b5871f2d09e89be6244bb4"
 CROS_WORKON_LOCALNAME="aosp/external/libmojo"
 CROS_WORKON_BLACKLIST="1"
```
```
Packages failed:
        dev-util/dbus-spy-0.0.1-r9
        net-libs/libmbim-1.19.0-r56
        dev-python/btsocket-1.0-r16
        app-benchmarks/punybench-0.0.1-r67
        media-libs/cros-camera-libcamera_exif-0.0.1-r136
        chromeos-base/screenshot-0.0.1-r70
        chromeos-base/vpd-0.0.1-r123
        chromeos-base/autotest-deps-p2p-0.0.1-r4803
        dev-util/turbostat-4.14.94-r2078
        media-libs/cros-camera-libcamera_client-0.0.1-r104
        media-libs/libmtp-0.0.1-r30
        chromeos-base/touchbot-1.0-r129
        x11-drivers/opengles-headers-0.0.1-r31
        sys-apps/rootdev-0.0.1-r33
        dev-util/libc-bench-0.0.1-r8
        chromeos-base/libweave-0.0.1-r1084
        dev-util/memory-eater-locked-0.0.1-r3
        chromeos-base/touch_firmware_test-0.0.1-r107
        dev-util/bsdiff-4.3.1-r20
        app-benchmarks/microbenchmarks-0.0.1-r4
        chromeos-base/gestures-conf-0.0.1-r91
        chromeos-base/libscrypt-1.1.6-r14
        chromeos-base/chromeos-storage-info-0.0.1-r79
        media-libs/webrtc-apm-0.0.1-r10
        chromeos-base/verity-0.0.1-r110
        dev-rust/9s-0.1.0-r3
        chromeos-base/chromiumos-assets-0.0.1-r12
        net-misc/tlsdate-0.0.5-r67
        dev-libs/dbus-c++-0.0.2-r62
        net-wireless/ath3k-1-r1
        chromeos-base/libevdev-0.0.1-r69
        media-libs/libsync-0.0.1-r3
```
My build has failed.

## Next


