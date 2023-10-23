# ChromiumOS-Build

## Date
2023/10/21
## Dev Env Info
### Hardware

AMD Ryzen 5 3600

32G DDR4 3200

2T NVMe SSD

Transparent Proxy in OpenWrt
### Software

win11 inside + wsl2 + ubuntu22
```
PS C:\Users\admin> wsl --version
WSL 版本： 1.3.14.0
内核版本： 5.15.90.3-1
WSLg 版本： 1.0.55
MSRDC 版本： 1.2.4419
Direct3D 版本： 1.608.2-61064218
DXCore 版本： 10.0.25880.1000-230602-1350.main
Windows 版本： 10.0.25977.1000
```

## Prepare
### Enable wsl systemd
see [WSL 中的高级设置配置 | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/wsl/wsl-config#systemd-support)

若要启用 systemd，请使用 sudo 通过管理员权限在文本编辑器中打开 wsl.conf 文件，并将以下行添加到 /etc/wsl.conf：

```
[boot]
systemd=true
```

### Install kvm
`sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager`

### Install docker
see [WSL 上的 Docker 容器入门 | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows/wsl/tutorials/wsl-containers)

## Build ChromiumOS
Follow [Chromium OS Developer Guide](https://chromium.googlesource.com/chromiumos/docs/+/master/developer_guide.md)

### Problems encountered
#### 1. init repo with certain branch

use `repo init -u https://chromium.googlesource.com/chromiumos/manifest -b release-R114-15437.B`

#### 2. network issue

retry `repo sync`  with global proxy

#### 3. create chroot

- error: `  File "/home/zhao/chromiumos/chromite/lib/osutils.py", line 1823, in IterateMountPoints
    source, destination, filesystem, options, _, _ = [
ValueError: too many values to unpack (expected 6)`
- reason: wsl mount path has blank space
- solution: add try catch

#### 4. build packages
`build_packages --board=${BOARD}`
- first try: 
  ```
  * ERROR: chromeos-base/chrome-binary-tests-0.0.1-r25::chromiumos failed (install phase):
    doexe: /build/amd64-generic/usr/local/build/autotest/client/deps/chrome_test/test_src/out/Release/wayland_hdr_client does not exist
  ```

- second try: 
  ```
  !!! Multiple package instances within a single package slot have been pulled
  !!! into the dependency graph, resulting in a slot conflict:

  You may want to try a larger value of
  the --backtrack option, such as --backtrack=30, in order to see if
  that will solve this conflict automatically.
  ```

- third try with `--backtrack=30`, result in success

#### 5. google api keys(not fixed yet)
I put my api keys in home directory called “.googleapikeys”, but it seems that the build system can’t find it. When start the VM, it shows missing “google api key”.

## Build kernel

```shell
# checkout kernel
cd ~/chromiumos/src/third_party/kernel/v5.15
git checkout v5.15.111
cd -
# build kernel
cros_workon --board amd64-generic start chromeos-kernel-5_15
FEATURES="noclean" cros_workon_make --board=${BOARD} --install chromeos-kernel-5_15
# update kernel
(cr) (release-R114-15437.B/(fbb4eabf...)) zhao@DESKTOP-ZHAO ~/chromiumos/src/scripts $ ./update_kernel.sh  --ssh_port 9222 --remote=localhost
15:55:27.352 INFO    : Target reports board is amd64-generic
15:55:27.355 INFO    : Target reports arch is x86
15:55:27.366 INFO    : Target reports root device is /dev/sda
15:55:27.387 INFO    : System is not using verity: updating firmware and modules
15:55:27.580 INFO    : Cleaning /boot, copying syslinux and /boot
15:55:28.474 INFO    : Cleaning and copying modules
15:55:30.072 INFO    : Copying vboot kernel image
4697+0 records in
4697+0 records out
19238912 bytes (19 MB, 18 MiB) copied, 0.501438 s, 38.4 MB/s
15:55:30.992 INFO    : Rebooting localhost...

     49s: can not connect to device
     53s: can not connect to device
     54s: reboot complete
15:56:25.313 INFO    : old kernel: 5.15.108-18907-gba143be42d3a #1 SMP PREEMPT Mon Oct 23 01:07:13 CST 2023
15:56:25.315 INFO    : new kernel: 5.15.111 #2 SMP PREEMPT Mon Oct 23 15:07:32 CST 2023
(cr) (release-R114-15437.B/(fbb4eabf...)) zhao@DESKTOP-ZHAO ~/chromiumos/src/scripts $ ssh -p 9222 root@localhost
Password:
localhost ~ # uname -a
Linux localhost 5.15.111 #2 SMP PREEMPT Mon Oct 23 15:07:32 CST 2023 x86_64 Intel Core Processor (Haswell, no TSX) AuthenticAMD GNU/Linux
```

## CrOS devserver
I didn't find docker in documentation, so I just use 'start_devserver' to start a devserver. But I encountered some problems when update payload, and I didn't find a solution yet.

