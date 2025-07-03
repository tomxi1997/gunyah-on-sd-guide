# Run gunyah vm with pvmfw

Snapdragon 8 elite device can avoid using pvmfw because it has proper implementation for `--protected-vm-without-firmware`. So for 8 elite devices, you can use [this one](https://github.com/polygraphene/gunyah-on-sd-guide) instead of using this page.
But for 8 gen 2 or 8 gen 3, the option doesn't work and you must stick with `--protected-vm`.

`--protected-vm` will execute pvmfw as the first code on the launched VM. We need some steps to run custom kernel and rootfs in this mode because pvmfw requires paticular format of kernel and disks.

This guide show you how to setup kernel image and disk image for pvmfw.

## Environment

1. Lenovo Legion Tablet Y700 gen4
2. Snapdragon 8 elite
3. Android 15
4. ZUXOS\_1.1.11.044\_250524\_PRC
5. Unlocked bootloader
6. root with APatch

It might work on other devices with gunyah hypervisor. I would appreciate it if you could leave a comment with the results if you try it.

## Instruction

### 1. Create dummy files to pass verificaiton

When run crosvm with protected VM mode, the first code to run in the VM will be pvmfw image.
The pvmfw verifies parameters passed from crosvm. We need to pass those checks if you don't want to patch pvmfw.

Run following command on termux or desktop linux to generate disk1.img and vm\_dt\_overlay.dtbo.

```
$ pkg i root-repo
$ pkg i gptfdisk dtc
$ dd if=/dev/zero of=disk1.img bs=1k count=1k
$ sgdisk -n '' disk1.img
$ sgdisk -c 1:vm-instance disk1.img
$ gdisk -l disk1.img
# Make sure the first sector = 34

$ echo -ne "Android-VM-instance\x01\x00" | dd of=disk1.img bs=512 seek=34 conv=notrunc

# The VM modifies disk1.img everytime you run the VM.
# Copy it from the backup everytime you run VM.
$ mv disk1.img disk1.img-org

$ dtc -I dts -O dtb > vm_dt_overlay.dtbo <<EOF
/dts-v1/;
/ {
  fragment@0 {
    target-path = [2f 00];
    __overlay__ {
      avf {
        untrusted {
          instance-id = <0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678 0x12345678>;
        };
      };
    };
  };
};
EOF
```

Then copy those files to `/data/local/tmp`.

### 2. Execute crosvm
Open root shell on the device, then:
```
# cd /data/local/tmp
# ulimit -l unlimited
# cp disk1.img-org disk1.img
# /apex/com.android.virt/bin/crosvm --log-level debug run \
  --disable-sandbox --no-balloon --protected-vm --swiotlb 14 \
  --params '' -i /apex/com.android.virt/etc/microdroid_initrd_debuggable.img \
  --mem 4096 --cpus 4 --device-tree-overlay vm_dt_overlay.dtbo --rwdisk disk1.img \
  /apex/com.android.virt/etc/fs/microdroid_kernel
[2025-06-25T11:33:49.864629795+00:00 DEBUG crosvm::crosvm::sys::linux] creating hypervisor: Gunyah { device: Some("/dev/gunyah") }
[2025-06-25T11:33:49.864895420+00:00 INFO  crosvm::crosvm::sys::linux::device_helpers] Trying to attach block device: disk1.img
[2025-06-25T11:33:49.864927139+00:00 INFO  disk] disk size 1048576,
[2025-06-25T11:33:49.864950993+00:00 INFO  disk] Disk image file is hosted on file system type f2f52010
[INFO] pvmfw config version: 1.0
[INFO] pVM firmware
avb_slot_verify.c:443: ERROR: initrd_normal: Hash of data does not match digest in descriptor.
[INFO] Successfully verified a debuggable payload.
[INFO] Please disregard any previous libavb ERROR about initrd_normal.
[INFO] Fallback to instance.img based rollback checks
[INFO] config: 0x2002000
[INFO] found a block device of size 1024KB
[INFO] No debug policy found.
[INFO] Starting payload...
[INFO] Expecting a bug making MMIO_GUARD_UNMAP return NOT_SUPPORTED on success
[    0.000000][    T0] Booting Linux on physical CPU 0x0000000000 [0x000f0480]
[    0.000000][    T0] Linux version 6.6.30-android15-5-g2485db222497-ab11868669 (kleaf@build-host) (Android (11368308, +pgo, +bolt, +lto, +mlgo, based on r510928) clang version 18.0.0 (https://android.googlesource.com/toolchain/llvm-project 477610d4d0d988e69dbc3fae4fe86bff3f07f2b5), LLD 18.0.0) #1 SMP PREEMPT Tue May 21 12:52:48 UTC 2024
[    0.000000][    T0] KASLR enabled
[    0.000000][    T0] Machine model: linux,dummy-virt
[    0.000000][    T0] stackdepot: disabled
[    0.000000][    T0] OF: reserved mem: 0x000000007fe17000..0x000000007fe17fff (4 KiB) nomap non-reusable dice
[    0.000000][    T0] software IO TLB: Reserved memory: created restricted DMA pool at 0x0000000180000000, size 14 MiB
[    0.000000][    T0] OF: reserved mem: initialized node restricted_dma_reserved, compatible id restricted-dma-pool
[    0.000000][    T0] OF: reserved mem: 0x0000000180000000..0x0000000180dfffff (14336 KiB) map non-reusable restricted_dma_reserved
...
[    0.432794][    T1] init: init first stage started!
[    0.433597][    T1] init: Unable to open /lib/modules, skipping module loading.
[    0.434921][    T1] init: Switching root to '/first_stage_ramdisk'
[    0.436153][    T1] init: [libfstab] Using Android DT directory /proc/device-tree/firmware/android/
[    0.445546][    T1] init: bool android::init::BlockDevInitializer::InitDevices(std::set<std::string>): partition(s) not found in /sys, waiting for their uevent(s): super, vbmeta_a
[   10.461045][    T1] init: Wait for partitions returned after 10013ms
[   10.463117][    T1] init: bool android::init::BlockDevInitializer::InitDevices(std::set<std::string>): partition(s) not found after polling timeout: super, vbmeta_a
[   10.468462][    T1] init: Failed to create devices required for first stage mount
[   10.470329][    T1] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00007f00
...
[2025-06-25T11:34:01.415320259+00:00 INFO  hypervisor::gunyah] exit type 2
[2025-06-25T11:34:01.415410884+00:00 INFO  crosvm::crosvm::sys::linux::vcpu] system reset event
[2025-06-25T11:34:02.021509790+00:00 INFO  crosvm] exiting with success
```

It can run the kernel, but the init stops because no proper disks are specified.

You could investigate how the disk is created, for example by examining the behavior of the `vm run-microdroid --protected` command, but in my case, I shifted toward running Debian with a custom kernel.

### 3. Compile kernel
Download source code of linux kernel 6.15.3 (latest version as of writing it).
I would recommend to use Linux PC (or WSL) to build kernel. Might works on Termux, but should be a bit slow.
```
$ sudo apt install llvm-19 clang-19 lld-19 bison flex bc
$ wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.15.3.tar.xz
$ tar xvf linux-6.15.3.tar.xz
$ cd linux-6.15.3/
$ wget https://android.googlesource.com/kernel/common/+archive/refs/tags/android-15.0.0_r0.81/arch/arm64/configs.tar.gz
$ tar xvf configs.tar.gz microdroid_defconfig
$ cp microdroid_defconfig .config
$ ./scripts/config --set-str SERIAL_8250_RUNTIME_UARTS 4 \
-e CGROUPS \
-e CGROUP_CPUACCT \
-e CGROUP_DEBUG \
-e CGROUP_DEVICE \
-e CGROUP_DMEM \
-e CGROUP_FAVOR_DYNMODS \
-e CGROUP_FREEZER \
-e CGROUP_MISC \
-e CGROUP_PERF \
-e CGROUP_PIDS \
-e CGROUP_RDMA \
-e CGROUP_SCHED \
-e CGROUP_WRITEBACK \
-e DEVTMPFS
$ echo | make LLVM=-19 ARCH=arm64 -j10 Image
...
  SORTTAB vmlinux
  OBJCOPY arch/arm64/boot/Image
```
The kernel image will created on `arch/arm64/boot/Image` if you succeeded.

If you want to use initrd, you can create them. But I'm using dummy one.
```
# Generate dummy initrd
$ dd if=/dev/zero of=initrd bs=1k count=1
# Or
# If you want real initrd, put your initrd on r/*, then:
$ (cd r/; find | cpio -o -Hnewc | lz4 -l -12 --favor-decSpeed > ../initrd.lz4)
# You can append bootconfig, but it is optional.
$ cat initrd.lz4 bootconfig > initrd
# Or (No bootconfig)
$ mv initrd.lz4 initrd
```

### 4. Sign kernel
Lenovo's pvmfw embeds AOSP pub key to verify kernel. We must sign the kernel with that key.
You need avbtool. Use prebuilt one from [v0.0.1 releases](https://github.com/polygraphene/gunyah-on-sd-guide/releases/tag/v0.0.1) or build it from AOSP source code.Prebuilt one should work on Ubuntu 24.04.

```
$ wget https://android.googlesource.com/platform/external/avb/+archive/refs/tags/android-15.0.0_r36/test/data.tar.gz
$ tar xvf testkey_rsa4096.pem
$ key=testkey_rsa4096.pem
$ cp (kernel_src)/arch/arm64/boot/Image kernel

$ cp initrd initrd-tmp
$ avbtool add_hash_footer --image initrd-tmp --partition_name initrd_debug --dynamic_partition_size --key $key
$ avbtool add_hash_footer --algorithm SHA256_RSA4096 --image kernel --partition_name boot --dynamic_partition_size --include_descriptors_from_image initrd-tmp --key $key
$ rm initrd-tmp

# Check if avb footer is properly appended
$ avbtool extract_vbmeta_image --image kernel --output kernel-vbfooter-tmp
$ avbtool info_image --image kernel-vbfooter-tmp
Minimum libavb version:   1.0
Header Block:             256 bytes
Authentication Block:     576 bytes
Auxiliary Block:          1472 bytes
Public key (sha1):        2597c218aae470a130f61162feaae70afd97f011
Algorithm:                SHA256_RSA4096
Rollback Index:           0
Flags:                    0
Rollback Index Location:  0
Release String:           'avbtool 1.3.0'
Descriptors:
    Hash descriptor:
      Image Size:            14374920 bytes
      Hash Algorithm:        sha256
      Partition Name:        boot
      Salt:                  94c0f5fc24b7d5a5a7d4a875f148864e91ec1d2dc9c2f0d116fb2c90a0940402
      Digest:                dc1490faa2318c7abcd718378263d62d292ce8fa97bb85057eaa959f0d9ef1cf
      Flags:                 0
    Hash descriptor:
      Image Size:            1024 bytes
      Hash Algorithm:        sha256
      Partition Name:        initrd_debug
      Salt:                  2394e924a785b7b9332d69d4a4c2a09e86a04a8cc454f881058b385b0530112f
      Digest:                ff7b2007bbcaaa675a4fb9878a585f93e63a9b4441b02471f7476b02cb0b2148
      Flags:                 0

$ rm kernel-vbfooter-tmp
$ adb push kernel /data/local/tmp
```

### 5. Download debian root fs

Download debian root fs built by google for use with the [Terminal App](https://www.androidpolice.com/android-15-linux-terminal-app/).

```
$ wget https://dl.google.com/android/ferrochrome/3500000/aarch64/images.tar.gz
$ adb push images.tar.gz /data/local/tmp
$ adb shell
(android) $ cd /data/local/tmp
(android) $ tar xvf images.tar.gz root_part
```

### 6. Execute crosvm again for Debian

Execute in root shell of the device:
```
# cd /data/local/tmp
# ulimit -l unlimited
# /apex/com.android.virt/bin/crosvm --log-level debug run \
  --disable-sandbox --no-balloon --protected-vm --swiotlb 14 \
  --params 'root=/dev/vdb' \
  --mem 4096 --cpus 4 --device-tree-overlay vm_dt_overlay.dtbo --rwdisk disk1.img \
  --rwdisk root_part -i initrd kernel
[2025-06-25T17:16:05.902952638+00:00 DEBUG crosvm::crosvm::sys::linux] creating hypervisor: Gunyah { device: Some("/dev/gunyah") }
[2025-06-25T17:16:05.903347951+00:00 INFO  crosvm::crosvm::sys::linux::device_helpers] Trying to attach block device: disk1.img
[2025-06-25T17:16:05.903423367+00:00 INFO  disk] disk size 1048576,
[2025-06-25T17:16:05.903438524+00:00 INFO  disk] Disk image file is hosted on file system type f2f52010
[2025-06-25T17:16:05.903461076+00:00 INFO  crosvm::crosvm::sys::linux::device_helpers] Trying to attach block device: root_part
[2025-06-25T17:16:05.903468107+00:00 INFO  disk] disk size 6308216320,
[2025-06-25T17:16:05.903472013+00:00 INFO  disk] Disk image file is hosted on file system type f2f52010
[INFO] pvmfw config version: 1.0
[INFO] pVM firmware
..
[    0.499156][    T1] VFS: Mounted root (ext4 filesystem) readonly on device 254:16.
[    0.500154][    T1] Freeing unused kernel memory: 768K
[    0.500925][    T1] Run /sbin/init as init process
SELinux:  Could not open policy file <= /etc/selinux/targeted/policy/policy.34:  No such file or directory
[    0.616104][    T1] systemd[1]: Failed to find module 'autofs4'
[    0.665976][    T1] systemd[1]: systemd 252.31-1~deb12u1 running in system mode (+PAM +AUDIT +SELINUX +APPARMOR +IMA +SMACK +SECCOMP +GCRYPT -GNUTLS +OPENSSL +ACL +BLKID +CURL +ELFUTILS +FIDO2 +IDN2 -IDN +IPTC +KMOD +LIBCRYPTSETUP +LIBFDISK +PCRE2 -PWQUALITY +P11KIT +QRENCODE +TPM2 +BZIP2 +LZ4 +XZ +ZLIB +ZSTD -BPF_FRAMEWORK -XKBCOMMON +UTMP +SYSVINIT default-hierarchy=unified)
[    0.677721][    T1] systemd[1]: Detected architecture arm64.

Welcome to Debian GNU/Linux 12 (bookworm)!

[    0.684400][    T1] systemd[1]: No hostname configured, using default hostname.
[    0.685242][    T1] systemd[1]: Hostname set to <localhost>.
[    0.687363][    T1] systemd[1]: Initializing machine ID from random generator.
..
[FAILED] Failed to start systemd-ti…0m - Network Time Synchronization.
See 'systemctl status systemd-timesyncd.service' for details.
[  OK  ] Reached target time-set.target - System Time Set.
[  OK  ] Finished systemd-update-ut… - Record Runlevel Change in UTMP.
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or "exit"
to boot into default mode.
Press Enter for maintenance
(or press Control-D to continue):
Reloading system manager configuration
Starting default.target
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemPress Enter for maintenance
(or press Control-D to continue):
root@localhost:~# uname -a
Linux localhost 6.15.3 #10 SMP PREEMPT Wed Jun 25 17:12:44 UTC 2025 aarch64 GNU/Linux
root@localhost:~# cat /etc/debian_version
12.8
root@localhost:~#
```

It doesn't boot up in a completely clean state, but Debian does start up for the time being.
This root fs image contains some services for Terminal App. Those services should be disabled. Also needs proper fstab.

## FAQ

### `vm` command causes `Permission denied` error 
```
thread 'main' panicked at packages/modules/Virtualization/virtualizationmanager/src/main.rs:134:42:
Failed to remove memlock rlimit: Status(-5, EX_ILLEGAL_STATE): 'Permission denied (os error 13)'
Error: Failed to connect to VirtualizationService
```

If you encounter this error on `vm run` or `vm info` comamnds, you need to run following command (or disable SELinux):
```
magiskpolicy --live "allow virtualizationservice magisk process { setrlimit }"
```

---

### failed to initialize virtual machine Out of memory (os error 12)
When `crosvm` put following erorr:

```
[2025-06-25T13:13:35.669951209+00:00 DEBUG crosvm::crosvm::sys::linux] creating hypervisor: Gunyah { device: Some("/dev/gunyah") }
[2025-06-25T13:13:35.670223397+00:00 INFO  crosvm::crosvm::sys::linux::device_helpers] Trying to attach block device: disk1.img
[2025-06-25T13:13:35.670261053+00:00 INFO  disk] disk size 1048576,
[2025-06-25T13:13:35.670272564+00:00 INFO  disk] Disk image file is hosted on file system type f2f52010
[2025-06-25T13:13:35.681230845+00:00 ERROR crosvm] exiting with error 1: the architecture failed to build the vm

Caused by:
    failed to initialize virtual machine Out of memory (os error 12)
```

Check dmesg and if you see following logs,

```
[ 8846.526582] misc gunyah: Failed to allocate parcel for DTB: -12
[ 8846.526811] gunyah_rsc_mgr hypervisor:qcom,resource-manager-rpc@940e2b5e1215cfb9: RM rejected message 51000015. Error: 7
[ 8846.526815] misc gunyah: Failed to reclaim parcel: -22
[ 8846.526819] misc gunyah: Failed to reclaim firmware parcel: -22
```

run the following command before executing `crosvm`

```
ulimit -l unlimited
```

This error is caused by the following line.
https://github.com/OnePlusOSS/android_kernel_oneplus_sm8750/blob/8e66bb68fe6735e93ef5dd59c184cf6341b0098d/drivers/virt/gunyah/vm_mgr_mem.c#L510
`account_locked_vm` adds locked memory usage of calling process (crosvm). This is 64kB in android's default. It is too small and cause `Out of memory`.

When launch vm via `vm` command, `virtmgr` does the same call as `ulimit`.
https://cs.android.com/android/platform/superproject/main/+/main:packages/modules/Virtualization/android/virtmgr/src/main.rs;l=115;drc=198eb9768cf9b74aa5a345cdd89079b3e23c36b8

---

### Recorded code hash doesn't match

If you encounter the followin error,
```
[ERROR] Dice measurements do not match recorded entry. This may be because of update: Recorded code hash doesn't match
```

Re-generate (copy from backup) disk1.img. VM modifies disk1.img every time you run the VM.

---

### Why non protected VMs don't work?

I don't know, but it seems unsupported?

---

### I want larger --swiotlb

Need the following patch for pvmfw.
https://github.com/polygraphene/android_packages_modules_Virtualization/commit/090f59bd4d49f925fb81abe29aa7ab38caf4a2ce

And you might need this in case of PCI related errors.
https://github.com/polygraphene/android_packages_modules_Virtualization/commit/c6f1433428f6a326b3736a48d8a924cc5535d058

## Note

1. You can build your own pvmfw from AOSP source code. So for another solution, you can remove all verification of kernel and disk image by editing source code and build it.
2. [packages/modules/Virtualization/docs](https://cs.android.com/android/platform/superproject/main/+/main:packages/modules/Virtualization/docs/;drc=2cb8e7397b171e0eea0d0c16e099a004da157e80) contains many documents about AVF and pvmfw.

### How to build pvmfw

1. Download AOSP source code

2. Edit source codes under packages/modules/Virtualization

3. Build and install
```
m pvmfw_img
adb push out/target/product/*/system/etc/pvmfw.img /data/local/tmp
adb shell su -c 'dd if=/data/local/tmp/pvmfw.img of=/dev/block/by-name/pvmfw'`adb shell getprop ro.boot.slot_suffix`
adb reboot
```

## Links

The `Guest Image Signing` section explains kernel signing:
https://cs.android.com/android/platform/superproject/main/+/main:packages/modules/Virtualization/guest/pvmfw/README.md;drc=51ec9d137836e117f1895ce70c81f2e97954464b

