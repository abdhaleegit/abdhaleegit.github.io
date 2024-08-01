# Introduction:
This blog talks about two ways to install Linux OS on IBM Power baremetal(aka Power NV) system

- Manual Network boot (aka netboot) via petitboot
- Automated OS install using op-test framework

Note: Given steps are for Fedora39 install and same can be followed for RHEL/SLES install

# Prerequisites
* HTTP server: Download Fedora 39 iso image and mount the files into http server path http://9.1.2.3:81/repo/fedora/39/
* IPMI tool installed
* FSP/BMC IP and IPMI userid/password
* Enable IPMI login via ASMI page -> Security -> External Services Management -> enable IPMI
* Network configured
* Disk attached to install

# Steps to Install Manually

1. From terminal CLI connect to FSP/BMC via IPMI tool
   ```
   BMC servers
   $ ipmitool -I lanplus -U username -P password -H 9.a.b.c sol activate

   FSP servers
   $ ipmitool -I lanplus -P password -H 9.x.y.z sol activate
     [SOL Session operational.  Use ~? for help]
   ``` 
2. From ASM/BMC page, Reboot server

3. At IPMI console, fall back to Petitboot shell

4. Make sure there is a network configured Petitboot Menu > System Configurations > Static Network > Public IP > Netmask > Gateway > DNS > OK

5. Download the intial files vmlinuz and initrd to the petitboot shell
   ```
   $ wget http://9.1.2.3:81/repo/fedora/39/ppc/ppc64/initrd.img
   $ wget http://9.1.2.3:81/repo/fedora/39/ppc/ppc64/vmlinuz
   ```
6. Load kexec command to perform netboot
   ```
   kexec -l vmlinux --initrd=initrdimage --command-line "ip=ipaddress::gateway:netmask:hostname:interface:none inst.vnc inst.repo=http_repo_link nameserver=xxx ifname=interface:macaddress  

   $ kexec -l vmlinuz --initrd=initrd.img --command-line "ip=9.x.yy.zzz::9.3.2.1:255.255.255.0:ltc1.ibm.com:enP3p9s0f0:none inst.vnc inst.repo=http://9.1.2.3:81/repo/fedora/39/ nameserver=9.3.2.1 ifname=enP3p9s0f0:08:3a:88:17:86:88"
   ```
7. Perform kexec boot
   ```
   $ kexec -e
   ```
8. Wait for stage 2 kernel boot to load

```
anaconda 39.32.6-2.fc39 for Fedora 39 started.
 * installation log files are stored in /tmp during the installation
 * shell is available on TTY2 and in second TMUX pane (ctrl+b, then press 2)
 * when reporting a bug add logs from /tmp as separate text/plain attachments
15:19:57 Starting VNC...
15:19:59 The VNC server is now running.
15:19:59

WARNING!!! VNC server running with NO PASSWORD!
You can use the inst.vncpassword=PASSWORD boot option
if you would like to secure the server.


15:19:59 Please manually connect your vnc client to ltc.ibm.com:1 (9.x.yy.zzz:1) to begin the install.

15:19:59 Attempting to start vncconfig

[anaconda]1:main* 2:shell  3:log  4:storage-log >Switch tab: Alt+Tab | Help: F1
```

Now connect to VNC server to complete installation


# Automated Linux OS install using op-test framework on Baremetal servers

Op-test is a python unit-test based test suite for validating IBM OpenPower boxes, which comprises many tests including booting host with multiple configurations etc.

Prerequisites:
- OS HTTP Repo http//dl.fedoraproject.org/pub/fedora-secondary/releases/28/Server/ppc64le/os/
- Kickstart file osimages/rhel/rhel.ks has the template for kickstart file, you can tune based on the packages that you want to install, for installing fedora make sure java is removed as it does not available in fedora repository.

It provides automated test suite to install Linux OS on baremetal systems  [op-test](https://github.com/open-power/op-test)

* op-test testcase to install Linux OS [testcase](https://github.com/open-power/op-test/blob/master/testcases/InstallRhel.py)

1. Clone op-test framework on your TP

   ```bash
   $ git clone https://github.com/open-power/op-test.git 
   $ op-test-framework
   ```

2. Create machine file matching to your server to be intalled

```
FSP base server:
---------------
[op-test]
bmc_type=FSP_PHYP
bmc_ip=ltc.stglabs.com  --> FSP ip/hostname
bmc_username=xyz --> FSP user
bmc_password=1111 -- > FSP password
bmc_usernameipmi=ADMIN --> IPMI userid
bmc_passwordipmi=ADMIN -- IPMI password
host_ip=9.x.y.zzz
host_user=user
host_password=1111
host_gateway=x.x.x.x
host_dns=x.x.x.x
host_mac=98:be:94:06:de:95 -->change as per your host
host_submask=255.255.255.0  -->change as per your host
host_scratch_disk=/dev/disk/by-id/scsi-35000c50098a05d6f  -->change as per your host
os_repo=https://dl.fedoraproject.org/pub/fedora-secondary/releases/28/Server/ppc64le/os/


BMC based server
----------------
[op-test]
bmc_type=OpenBMC
bmc_ip=x.x.x.x
bmc_username=ADMIN
bmc_password=ADMIN
bmc_usernameipmi=ADMIN
bmc_passwordipmi=ADMIN
host_ip=x.x.x.x
host_user=root
host_password=passw0rd
host_gateway=x.x.x.x
host_dns=x.x.x.x
host_mac=98:be:94:06:de:95 -->change as per your host
host_submask=255.255.255.0  -->change as per your host
host_scratch_disk=/dev/disk/by-id/scsi-35000c50098a05d6f  -->change as per your host
os_repo=https://dl.fedoraproject.org/pub/fedora-secondary/releases/28/Server/ppc64le/os/
```

* Now start the installation

    ```
    $./op-test -c machine.conf --run testcases.InstallRhel.InstallRhel
    ```
test complete like below

```
^[[32m[   39.218896] ^[[0m^[[33mtg3 0005:09:00.0 net0^[[0m: EEE is disabled
^[[32m[   39.218976] ^[[0m^[[33mIPv6^[[0m: ADDRCONF(NETDEV_CHANGE): net0: link becomes ready
[console-pexpect]#echo $?
0
/home/op-test-framework/test-reports/Kernel_dmesg_log_Thu_May_17_11:20:20_2018.log
OK (939.294s)

----------------------------------------------------------------------
Ran 1 test in 939.294s

OK

Generating XML reports...
Output written to: /home/op-test-framework/test-reports/*20180517110439*
```

Now you have IBM Power system with Fedora28 installed!!

Thanks for reading my blog, hope this has helped you install baremetal HOST OS !!!!
