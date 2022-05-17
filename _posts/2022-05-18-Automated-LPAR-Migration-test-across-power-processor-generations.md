# Introduction
The blog talks about our agile journey to enable op-test framework support for cross processor LPAR migration.

Op-test framework provides tools and API's for automated testing of Power systems, for any given power system test can be run from outside the system under test (SUT) and monitored remotely.

The current op-test implementation takes one HMC input details as part of system configuration and we need enhancement to make sure the op-test framework APIs are extended to support multiple HMC consoles and Develop test using the new API's to cover LPAR migration across HMC.

# Brief about LPM technology
Live Partition Mobility (LPM) is an IBM PowerVM feature to migrate Linux OS partition from one server to another server, It provides ability to migrate LPAR for server consolidation, workload balancing, evacuating servers for planned maintenance, migrating from older technology (POWER4 and above) to newer technology.

The LPM operation moves the LPAR dynamically i.e processor states, memory, virtual devices etc using Hardware management console, it supports both active Migration (Live Partition Mobility) i.e when LPAR is online and also inactive Migration (cold parititon mobitlity) when LPAR is offline.

The LPM process first validate configuration and create LPAR configuration on target server, recreates virtual resources on LPAR at target server and than Migrate the state of the LPAR in memory, Remove the old LPAR configuration from the source server and finally free up the old resource on the source server.

# HMC LPM commands
    
    * Validate LPM setup before actual migration
      $ migrlpar -o v -m source_system -t target_system -p lpar_name
     
    * Start LPAR Migration
      $ migrlpar -o m -m source_system -t target_system -p lpar_name

    * In case if you want to stop LPM in progress
      $ migrlpar -o s -m source_system -t target_system -p lpar_name

    * If the LPM failed and LPAR needs Recovery (run the command where lpar is seen)
      $ migrlpar -o r -m source_system -p lpar_name

    * To migrate back from destination to source
      $ migrlpar -o m -t source_system -m target_system -p lpar_name

    * In case if target server is in different remote HMC, then first authenticate the ssh keys
      $ mkauthkeys -u  hscroot --ip <source/destination hmc ip> --passwd password

    * Migrate LPAR to a server in remote HMC
      $ migrlpar -o m -m source_system -t target_system -p  lpar_name -u hscroot --ip <target hmc ip>

    * Migrate back from target hmc server to source hmc (run the command in remote hmc)
      $ migrlpar -o m -m target_system -t source_system -p  lpar_name -u hscroot --ip <source hmc ip>

# Current LPM automation coverage
The existing Avocado LPM tests [lpm.py](https://github.com/avocado-framework-tests/avocado-misc-tests/blob/master/io/net/virt-net/lpm.py) is run from inside SUT via ssh console and it cannot handle well when crash or fail

* LPM with SEA (shared ethernet adapter)
* LPM with SEA and vNIC as test device
* LPM with SEA and vSCSI
* LPM with SEA and vFC
* LPM with HNV and vETH as backup device
* LPM with HNV and vNIC as backup device

# Challenges with existing LPM automation
Currently, framework do not support Linux LPAR Migration test across different Power processor generation like Migration from Power 9 to Power 10 etc.

  1. Basic LPM test scripts with minimal coverage in avocado
  2. Test running inbound, i.e. runs from inside system under test
  3. No log collection in case of failure
  4. No remote HMC LPM support
  5. No LPAR recovery in case of LPM failures
  6. No support to migrate across processor generations

Op-test framework provides API's to run test remotely and LPAR migration process from source server to destination server is monitored remotely and results are collected back to local.

# Implementation as part of this project
  * Enable op-test Framework APIs to support LPAR migration across different Power processors
  * LPM Test development using the new APIs
  * OS, Phyp, VIOS Log collection at each stage of LPM
  * Extend tests to support LPM with vNIC and HNV SRIOV virtual devices

![](https://github.com/abdhaleegit/abdhaleegit.github.io/raw/master/resource/lpmflow.jpg)


# Implementation details
The LPM code implementation has 2 main blocks OpTestLPM_LocalHMC() for single HMC LPM and OpTestLPM_CrossHMC() for remote HMC LPM, a new target hmc instance allowes the LPM commands to be run from remote HMC.
Based on the user input configurations, test start with LPM prerequisites check like Moving Service Partition (MSP) on source and destination vios is enabled, RMC service is up on LPAR and HMC etc.

While the LPAR migration process is monitored and automated recovery will be performed in case of LPM failure to migrate LPAR and logs are being collected in parallel.

* OS logs collected at different LPM stages (pre-forward LPM, post-forward LPM, post-backward LPM) [code](https://github.com/open-power/op-test/pull/681)
    ```
    dmesg, journalctl -a, /var/log/messages, /var/log/boot.log
    (LPM-specific): /var/log/drmgr
    ```
* Proper dmesg log parsing for errors done at different LPM stages and When test fails sosreport tar copied to local machine 

* HMC logs collected (includes remote hmc logs too)
    ```
    lshmc -V, pedbg -c -q 4
    ```
* VIOS logs collected (includes remote vios logs too)
    ```
    /proc/version, ioslevel, errlog, snap
    ```

Only the default logs are collected all the time, and in case of failure snap logs and pedbug logs are collected along with default logs, finally all the logs are than copied back to the remote server from where the test were triggered

# LPM latest test coverage in op-test automation

1. Single HMC LPM: [code](https://github.com/open-power/op-test/blob/c3b945bb06ad875f487cba86eb1fc88823972e4c/testcases/OpTestLPM.py#L263)
 * with vNIC
 * with HNV SRIOV (veth as backup)
 * with HNV SRIOV (vNIC as backup)

2. Remote HMC LPM: [code](https://github.com/open-power/op-test/blob/c3b945bb06ad875f487cba86eb1fc88823972e4c/testcases/OpTestLPM.py#L320)
 * with vNIC
 * with HNV SRIOV (veth and vnic as backup)

3. Cross-processor LPM: [code](https://github.com/open-power/op-test/pull/671)
 * P9 to P8 
 * P9 to P10 etc

4. LPAR forward and backword migration in a loop for N iteration [code](https://github.com/open-power/op-test/pull/687)

5. LPAR migration while stress-ng workload is running [code](https://github.com/open-power/op-test/pull/687)


# How to run the automated LPM tests

Clone the op-test `git clone https://github.com/open-power/op-test`

How to run single HMC LPM test ?

* sample config file for local hmc test
```
[op-test]
bmc_type=FSP_PHYP
bmc_ip=9.x.x.x
bmc_username=aaa
bmc_password=bbb
host_ip=9.x.x.x
host_user=aaa
host_password=bbb
hmc_ip=9.x.x.x
hmc_username=aaa
hmc_password=bbb
system_name=xyz
lpar_name=lpar1
lpar_prof=default_profile
host_cmd_timeout=36000
machine_state=OS
target_system_name=abc
lpar_vios=xyz-vios1
vios_ip=9.x.x.x
vios_username=aaa
vios_password=aaa
```

command to run local hmc test
```bash
./op-test --config-file <config file name> --run testcases.OpTestLPM.OpTestLPM_LocalHMC
```

How to run remote HMC LPM test ?

* sample config file for remote hmc lpar
```
[op-test]
bmc_type=FSP_PHYP
bmc_ip=9.x.x.x
bmc_username=aaa
bmc_password=bbb
host_ip=9.x.x.x
host_user=aaa
host_password=bbb
hmc_ip=9.x.x.x
hmc_username=aaa
hmc_password=bbb
system_name=xyz
lpar_name=lpar1
lpar_prof=default_profile
host_cmd_timeout=36000
machine_state=OS
target_system_name=abc
target_hmc_ip=9.x.x.x
target_hmc_username=aaa
target_hmc_password=bbb
lpar_vios=xyz-vios1
vios_ip=9.x.x.x
vios_username=aaa
vios_password=aaa
remote_lpar_vios=abc-vios1
remote_vios_ip=9.x.x.x
remote_vios_username=aaa
remote_vios_password=aaa
```

Run the test
```bash
./op-test --config-file <config file name> --run testcases.OpTestLPM.OpTestLPM_CrossHMC
```

Sample config to run cross HMC LPM with HNV SRIOV test

```
[op-test]
bmc_type=FSP_PHYP
bmc_ip=9.x.x.x
bmc_username=aaa
bmc_password=bbb
host_ip=9.x.x.x
host_user=aaa
host_password=bbb
hmc_ip=9.x.x.x
hmc_username=aaa
hmc_password=bbb
system_name=xyz
lpar_name=lpar1
lpar_prof=default_profile
host_cmd_timeout=36000
machine_state=OS
target_system_name=abc
target_hmc_ip=9.x.x.x
target_hmc_username=aaa
target_hmc_password=bbb
lpar_vios=abc-vios1
vios_ip=9.x.x.x
vios_username=aaa
vios_password=bbb
remote_lpar_vios=abc-vios1
remote_vios_ip=9.x.x.x
remote_vios_username=aaa
remote_vios_password=bbb
options=--migsriov 1
```

Sample config to run local HMC with vNIC device

```
[op-test]
bmc_type=FSP_PHYP
bmc_ip=9.x.x.x
bmc_username=aaa
bmc_password=bbb
host_ip=9.x.x.x
host_user=aaa
host_password=bbb
hmc_ip=9.x.x.x
hmc_username=aaa
hmc_password=bbb
system_name=xyz
lpar_name=lpar1
lpar_prof=default_profile
host_cmd_timeout=36000
machine_state=OS
target_system_name=abc
lpar_vios=xyz-vios1
vios_ip=9.x.x.x
vios_username=aaa
vios_password=bbb
remote_lpar_vios=abc-vios1
remote_vios_ip=9.x.x.x
remote_vios_username=aaa
remote_vios_password=bbb
slot_num=3
bandwidth=2
adapters=U78D8.ND0.FGD0047-P0-C7-C0
target_adapters=U78D8.ND0.FGD002W-P0-C6-C0
ports=0
target_ports=0
options=--vniccfg 1
interface=env3
interface_ip=100.53.53.174
netmask=255.255.255.0
peer_ip=100.53.53.17
```

Sample config for LPM with N iterations

```
[op-test]
bmc_type=FSP_PHYP
bmc_ip=9.x.x.x
bmc_username=aaa
bmc_password=bbb
host_ip=9.x.x.x
host_user=aaa
host_password=bbb
hmc_ip=9.x.x.x
hmc_username=aaa
hmc_password=bbb
system_name=xyz
lpar_name=lpar1
lpar_prof=default_profile
host_cmd_timeout=36000
machine_state=OS
target_system_name=abc
lpar_vios=abc-vios1
vios_ip=9.x.x.x
vios_username=aaa
vios_password=bbb
remote_lpar_vios=abc-vios1
remote_vios_ip=9.x.x.x
remote_vios_username=aaa
remote_vios_password=bbb
iterations=2
```

Sample config for LPM test with stress-ng workload

```
[op-test]
bmc_type=FSP_PHYP
bmc_ip=9.x.x.x
bmc_username=aaa
bmc_password=bbb
host_ip=9.x.x.x
host_user=aaa
host_password=bbb
hmc_ip=9.x.x.x
hmc_username=aaa
hmc_password=bbb
system_name=xyz
lpar_name=lpar1
lpar_prof=default_profile
host_cmd_timeout=36000
machine_state=OS
target_system_name=abc
lpar_vios=abc-vios1
vios_ip=9.x.x.x
vios_username=aaa
vios_password=bbb
remote_lpar_vios=xyz-vios1
remote_vios_ip=9.x.x.x
remote_vios_username=aaa
remote_vios_password=bbb
stressng_command=stress-ng --cpu 32 --io 50 --vm 12 --vm-bytes 20G --fork 4 --timeout 30m
```

LPM tests source [code](https://github.com/open-power/op-test/blob/master/testcases/OpTestLPM.py)

# Summary
Live Partition Mobility (LPM) on power servers is a key deliverables for our customers, contineous development and test is happening at IBM to deliver LPM feature with good quality, with the implementation of new API's inplace in op-test framework, it can be leveraged by developement team for unit test and also can be enabled with end to end CI/CR framework for upstream and distributiontest, LPM test automation coverage can be extend to cover huge test matrix from FVT/SVT/ISST teams around LPAR migration and other key test areas

Right from origin of the idea to implementation, agile methodoligies was employed throught out the journey, with contineous involvement of developement team in defining the requirements and timely feedback has helped align ourselves to the final goal, also adopting to new ways of working like leveraging Trello/Jira wherein broader goals were formulated into smaller tasks and than each tasks are achieved in a predefined sprints.

Special thanks to **Rashi Jhawar** for developing code to enable LPM support on op-test, despite of remote working challenges she picked up skills quickly and delivered the project with good quality in time, also I take this opportunity to thank all the team members involved **Sachin**, **Pavithra**, **Iranna**, **Manu**, **Preeti**, **Praveen**, **Vaishnavi** et all. for being part of this project to make it successful


Thank you for reading my blog!!
