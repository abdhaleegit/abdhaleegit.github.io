# Introduction:
A step by step guide to configure KDUMP on Upstream kernel, unlike distro kernels make use of inbuilt kdump tools such as kexec and makedumpfile, Upstream kdump requires latest binaries to be built to support upstream kernels. The blog also talks about automated kdump test validation using openpower test framework and debug trace using crash tool


# Prerequisites
* Install RHEL8.x distribution on your PowerVM LPAR 
* KDUMP configuration steps given below may differ for non Redhat OS
    1. Reserve crash kernel memory example `crashkernel=1024M` in `/etc/default/grub`
    2. Update grub.cfg with command `grub2-mkconfig -o /boot/grub2/grub.cfg`
    3. Reboot the OS for the crash kernel settings to take an effect
    4. Run `dmesg | grep Res` or `cat /proc/cmdline` commands to verify the crashkernel memory reserverd
    5. Edit `/etc/kdump.conf` and set kdump target path example `path /var/crash`
    6. Restart kdump service to make dump configuration effective `service kdump restart`

# Compile and install upstream kernel

* Download the latest upstream kernel source from [here](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git) 
* Copy the distro config from /boot and make sure below config options enabled  

1. Enable debuginfo option to build kernel with debug symbols, as vmlinux enabled with debuginfo is required to analyze the vmcore dump file.
  ```
  CONFIG_DEBUG_INFO=y
  # CONFIG_DEBUG_INFO_REDUCED is not set
  # CONFIG_DEBUG_INFO_SPLIT is not set
  # CONFIG_DEBUG_INFO_COMPRESSED is not set
  CONFIG_DEBUG_INFO_DWARF4=y
  ```
  
2. Enable "kexec system call" in "Processor type and features."
   ```
   CONFIG_KEXEC=y
   ```
3. Enable "sysfs file system support" in "Filesystem" -> "Pseudo filesystems." 
   This is usually enabled by default.
   ```
   CONFIG_SYSFS=y
   ```
4. Enable "Build a kdump crash kernel" support under "Kernel" options:
   ```
   CONFIG_CRASH_DUMP=y
   ```
5. Enable "Build a relocatable kernel" support
   ```
   CONFIG_RELOCATABLE=y
   ```

* Linux Build and install

  ```bash
  make olddefconfig
  make -j$(nproc) && make -j modules && make -j modules_install && make install
  ```
* Reboot the LPAR to boot to the newly installed upstream kernel

# Build and install upstream kdump binaries

Upstream kernel requires updated kdump binaries as distro provided makedumpfile fails to collect dump with upstream kernel.

* Known issue with makedumpfile on upstream kernel
    ```
    The kernel version is not supported.
    The makedumpfile operation may be incomplete.
    check_release: Can't get the kernel version.
    makedumpfile Failed.
    Running makedumpfile --dump-dmesg /proc/vmcore failed (1).
    ```
* lets build and replace the distro provided makedumpfile and crash tools

  * makedumpfile
     ```bash
     git clone https://github.com/makedumpfile/makedumpfile.git
     yum install ncurses-devel zlib-devel bison bzip2-devel lzo lzo-devel snappy-devel elfutils-devel 
     cd makedumpfile; make LINKTYPE=dynamic USELZO=on USESNAPPY=on ;make install
     ```
  * crash
    ```bash
    git clone https://github.com/crash-utility/crash.git
    cd crash; make USELZO=on USESNAPPY=on; make install
    ```
  * kexec, vmcore-dmesg
     ```bash
     git clone https://git.kernel.org/pub/scm/utils/kernel/kexec/kexec-tools.git
     cd kexec-tools;./bootstrap;./configure;make all;make install
     ```
  * Rebuild initrd and reboot for latest installed binaries to take effect
     ```bash
     dracut -f
     reboot back to upstream kernel
     ```
* Verify the kdump service is running `service kdump status`
* Now login and trigger the crash `echo 1 > /proc/sys/kernel/sysrq ; echo c > /proc/sysq-trigger`
* KDUMP kernel gets booted and vmcore is collected at /var/crash 
```
$ ls -lrt /var/crash/127.0.0.1-2021-08-27-09:56:46
total 114164
-rw-r--r-- 1 root root     22906 Aug 27 09:56 vmcore-dmesg.txt
-rw------- 1 root root 116941747 Aug 27 09:56 vmcore
-rw-r--r-- 1 root root     32551 Aug 27 09:56 kexec-dmesg.log
```

# Analyze core dump using crash utility
Dump analysis using crash utility requires vmcore and upstream vmlinux (enabled with debug symbols)

```
crash /boot/vmlinuz-5.14.0-autotest /var/crash/vmcore
```

known issue while crash analysis

```bash
'crash: compressed kdump: uncompress failed: no lzo compression support'
fix this by running `make lzo; make install` from the crash source

though CONFIG_DEBUG_INFO=y config was enabled
'crash: /boot/vmlinuz-5.18.0-rc6 no debugging data available'
make sure CONFIG_DEBUG_INFO_REDUCED is not set, else you might los debug data
```

* Refer to kdump documentation for more details [kdump](https://www.kernel.org/doc/Documentation/kdump/kdump.txt)

# Automated upstream kernel install and kdump test run using op-test 
OpenPower Test Framework provides automated test suite to install upstream kernel and run the kdump tests [op-test](https://github.com/open-power/op-test)

* op-test command to install upstream kernel [testcase](https://github.com/open-power/op-test/blob/master/testcases/InstallUpstreamKernel.py)

   ```bash
   git clone https://github.com/open-power/op-test.git 

   ./op-test --run testcases.InstallUpstreamKernel.InstallUpstreamKernel --git-repo=https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git --git-branch=master --git-repoconfigpath=kernelconfig -c ./machine.conf
   ```
where as machine.conf file will look like

```
[op-test]
bmc_type=FSP_PHYP
bmc_ip=ltc.stglabs.com
bmc_username=xyz
bmc_password=1111
host_ip=9.x.y.zzz
host_user=user
host_password=1111
hmc_ip=hmc.labs.ibm.com
hmc_username=hmcuser
hmc_password=hmcpwd
system_name=ltc1
lpar_name=CI-ltc1
lpar_prof=default_profile
host_cmd_timeout=7200
git_home=/home/linux_src
use_kexec=True
machine_state=OS
```

* op-test command to run kdump test [testcase](https://github.com/open-power/op-test/blob/master/testcases/PowerNVDump.py)
   ```bash
    ./op-test --run-suite osdump-suite -c machine_test.conf
   ```
where as the machine_test.conf will look like
```
[op-test]
bmc_type=FSP_PHYP
bmc_ip=ltc.stglabs.com
bmc_username=xyz
bmc_password=1111
host_ip=9.x.y.zzz
host_user=user
host_password=1111
hmc_ip=hmc.labs.ibm.com
hmc_username=hmcuser
hmc_password=hmcpwd
system_name=ltc1
lpar_name=CI-ltc1
lpar_prof=default_profile
host_cmd_timeout=36000
machine_state=OS
dump_server_ip=9.y.zz.xx
dump_server_pw=password
dump_path=/mnt
```
The kdump test suite covers kdump over local, dump over ssh and so on.

Thanks for reading my blog, hope this has helped you setting KDUMP over upstream kernel
