---
layout: post
title: Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 2
summary: Here is about SNMP O_O 
featured-img:
categories: Linux Monitoring Kubernetes 
tags: [ grafana, prometheus, loki, kubernetes, notes, linux ]
---
- [Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 1. Kubernetes cluster](https://timeforplanb123.github.io/k8s-monitoring-part-one-k8s-cluster/)
- [Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 2. SNMP](https://timeforplanb123.github.io/k8s-monitoring-part-two-snmp/)

SNMP in 2023? 
There is nothing more permanent than temporary. Originally conceived as a temporary protocol, SNMP firmly settled in the world of network devices. Yes, in 2023, SNMP has not lost its relevance. Before talking about Prometheus SNMP Exporter and Prometheus SNMP Config Generator, I would like to write a simple note on how to search SNMP OIDs.


## How to search SNMP OIDs for monitoring

### Prepare the host

The host here is Ubuntu Desktop. I'll prepare it for working with SNMP and OIDs searching:

1. Install a standard set of MIBs on host. Vendors' MIBs are based on these MIBs:
```text
sudo apt install snmp-mibs-downloader
```
Now the default MIBs are installed in `/usr/share/snmp/mibs/` directory. Here are IF-MIB, SNMPv2-MIB and many others. For example, IF-MIB, as a rule, is a universal module and contains the OIDs with network interface parameters (here are interface name, mtu, description, mac-address, etc.). For most network devices, this MIB is enough to get the necessary data.
2. Install the CLI utilities for working with MIB and SNMP (snmpwalk, snmpget, snmptranslate, snmptable):
```text
sudo apt install snmp
```
3. Find any MIB browser. Ireasoning is ok, there are app for all platforms - [https://www.easysnmp.com/tools/snmp-browsers/](https://www.easysnmp.com/tools/snmp-browsers){:target="_blank"}
MIB browser will provide a convenient search through the MIB trees.


### How to search

1. Find the MIB for interested network device:
- public MIBs can be found on the vendor's resources. In addition to the MIB, there can be references with the MIB tree structure, OID descriptions, public MIBs, MIB browser usage examples.
- for example, here - [http://www.circitor.fr/Mibs/Mibs.php](http://www.circitor.fr/Mibs/Mibs.php){:target="_blank"} . From here I downloaded HUAWEI-MIB - `~/.snmp/mibs$ curl http://www.circitor.fr/Mibs/Mib/H/HUAWEI-MIB.mib > HUAWEI-MIB.mib`  
2. Open the downloaded MIB, for example, `HUAWEI-MIB.mib` with any text editor, look at the dependency on other MIBs:
```text
HUAWEI-MIB DEFINITIONS ::= BEGIN

    IMPORTS
        enterprises, MODULE-IDENTITY
            FROM SNMPv2-SMI;
```
`HUAWEI-MIB.mib` takes a piece of MIB tree (usualy these are the first OID digits, for example, `iso(1) org(3) dd(6) internet(1) private(4) enterprises(1)`) from `SNMPv2-SMI.mib`. It's important to understand here that, `HUAWEI-MIB.mib` is a limited set of OIDs, and other OIDs may be associated with other MIBs. For example, interface counters are collected in the default IF-MIB, but IF-MIB and HUAWEI-MIB are not related in any way, there are no `IMPORTS`).
3. And now the process is a bit creative. I upload the necessary MIBs to the MIB browser, open MIB references and a terminal with snmpwalk + snmpget + snmptranslate. Then I search suitable OIDs by name, description in MIB browser.
In addition to manual search via MIB browser, you can use snmptranslate as an MIB browser alternative or auxiliary utility. For example: 
- **expand some MIB tree:**
```text
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -Ln -Tp HUAWEI-MIB::hwDatacomm.42.2.1.16.1.2
+--hwDatacomm(25)
   |
   +--hwBRASMib(40)
```
or by the OID name:
```text
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -Ln -Tp HUAWEI-MIB:hwDatacomm
+--hwDatacomm(25)
   |
   +--hwBRASMib(40)
```
- **search some OIDs:**  
```text
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -Ln -Td HUAWEI-MIB::hwDatacomm.42.2.1.16.1.2
HUAWEI-MIB::hwDatacomm.42.2.1.16.1.2
hwDatacomm OBJECT-TYPE
  -- FROM   HUAWEI-MIB
::= { iso(1) org(3) dod(6) internet(1) private(4) enterprises(1) huawei(2011) huaweiMgmt(5) hwDatacomm(25) 42 2 1 16 1 2 }
```
```text
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -Ln -Td HUAWEI-MIB:hwVlan
HUAWEI-MIB::hwVlan
hwVlan OBJECT-TYPE
  -- FROM   HUAWEI-MIB
::= { iso(1) org(3) dod(6) internet(1) private(4) enterprises(1) huawei(2011) huaweiMgmt(5) 6 }
```
```text
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -Ln -On HUAWEI-MIB:hwVlan
.1.3.6.1.4.1.2011.5.6
```
```text
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -Ln -Of HUAWEI-MIB:hwVlan
.iso.org.dod.internet.private.enterprises.huawei.huaweiMgmt.hwVlan
```
- **search MIB by OID:**
```text
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -Ln -OS 1.3.6.1.4.1.2011.5.25.31.1.1.1.1.5
HUAWEI-MIB::hwDatacomm.31.1.1.1.1.5
```
And, of course, all options are described in the `snmptranslate --help`.  Here only the ones used above:
  `-M` - directories with MIBs (`./` - the current directory with `HUAWEI-MIB.mib`, `/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana` - directories with loaded default MIBs).  
  `-Ln` - don't sow logs on stdout, thus don't show the MIB hierarchy errors.  
  `-Td` - `-T` option is a set various options controlling report produced. There are very useful options here. For example `-Td` prints full OID details, including MIB name + any MIB data described given OID (MAX-ACCESS, STATUS, DESCRIPTION). The `-Td`, `-Tp`, `-Tdp` overlap each other, the last one in the chain is used.  
  `-Onf` - `-O` option toggles the various defaults that control the output display. `-Onf` shows OID numbers (`n`), show all OIDs (`f`). `f` flag overrides `n` flag here.  
  `-OS` - print MIB module-id plus last element  
  `-m` - specify list of MIB or all MIBs with `ALL` value
- **if there are loaded MIBs in the current directory for different network devices, then you can check for the same OID:**
```text
snmptranslate -M ./ -m ALL -Ts -Ln | grep OID 
```
- **search MIB objects by regular expressions(`-TB` option):**  
I downloaded HUAWEI-ENTITY-EXTENT-MIB for this example - `~/.snmp/mibs$ curl http://www.circitor.fr/Mibs/Mib/H/HUAWEI-ENTITY-EXTENT-MIB.mib > HUAWEI-ENTITY-EXTENT-MIB.mib`.  
```text
# the value of the `-M` option should be the directory with HUAWEI-ENTITY-EXTENT-MIB (./) and with all imported MIB (/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana)
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -m HUAWEI-ENTITY-EXTENT-MIB -TB hwEntityMem
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemUsed
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemoryAvgUsage
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemoryType
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemSizeMega
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemSize
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemUsageThreshold
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemUsage
```
```text
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -m HUAWEI-ENTITY-EXTENT-MIB -TB hwEntityMem*
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemUsed
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemoryAvgUsage
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemoryType
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemSizeMega
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemSize
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemUsageThreshold
HUAWEI-ENTITY-EXTENT-MIB::hwEntityMemUsage
```
```text
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -m HUAWEI-ENTITY-EXTENT-MIB -On -TB hwEntityMem*
.1.3.6.1.4.1.2011.5.25.31.1.1.1.1.37
.1.3.6.1.4.1.2011.5.25.31.1.1.1.1.36
.1.3.6.1.4.1.2011.5.25.31.1.1.1.1.31
.1.3.6.1.4.1.2011.5.25.31.1.1.1.1.19
.1.3.6.1.4.1.2011.5.25.31.1.1.1.1.9
.1.3.6.1.4.1.2011.5.25.31.1.1.1.1.8
.1.3.6.1.4.1.2011.5.25.31.1.1.1.1.7
```
And the same things, but in detail from MIB (`d` option):
```text
~/.snmp/mibs$ snmptranslate -M ./:/usr/share/snmp/mibs/ietf:/usr/share/snmp/mibs/iana -m HUAWEI-ENTITY-EXTENT-MIB -On -TBd hwEntityMem*.1.3.6.1.4.1.2011.5.25.31.1.1.1.1.37
hwEntityMemUsed OBJECT-TYPE
  -- FROM	HUAWEI-ENTITY-EXTENT-MIB
  SYNTAX	Unsigned32
  MAX-ACCESS	read-only
  STATUS	current
  DESCRIPTION	"The memory information for the entity. This object indicates how
                many bytes in the memory have been used. "
::= { iso(1) org(3) dod(6) internet(1) private(4) enterprises(1) huawei(2011) huaweiMgmt(5) hwDatacomm(25) hwEntityExtentMIB(31) hwEntityExtObjects(1) hwEntityState(1) hwEntityStateTable(1) hwEntityStateEntry(1) 37 }
.1.3.6.1.4.1.2011.5.25.31.1.1.1.1.36
hwEntityMemoryAvgUsage OBJECT-TYPE
  -- FROM	HUAWEI-ENTITY-EXTENT-MIB
  SYNTAX	Integer32 (0..100)
  MAX-ACCESS	read-only
  STATUS	current
  DESCRIPTION	"Average memory usage within a specified statistical interval."
::= { iso(1) org(3) dod(6) internet(1) private(4) enterprises(1) huawei(2011) huaweiMgmt(5) hwDatacomm(25) hwEntityExtentMIB(31) hwEntityExtObjects(1) hwEntityState(1) hwEntityStateTable(1) hwEntityStateEntry(1) 36 }
...
```


## snmpwalk and snmpget

To check the found OIDs on the network device, you can use the `snmpwalk` and `snmpget` utilities. All flags used in `snmptranslate`, except `-T`, are relevant here.

Using HUAWEI-MIB, I'll check some OIDs on test huawei switch (I have `HUAWEI-MIB.mib` in the current directory):
```text
~/.snmp/mibs$ snmpwalk -v2c -c public -Ln -On 192.168.0.3 hwAaa
.1.3.6.1.4.1.2011.5.2.1.1.1.1.6.114.97.100.105.117.115 = STRING: "radius"
.1.3.6.1.4.1.2011.5.2.1.1.1.1.7.100.101.102.97.117.108.116 = STRING: "default"
.1.3.6.1.4.1.2011.5.2.1.1.1.2.6.114.97.100.105.117.115 = INTEGER: 3
.1.3.6.1.4.1.2011.5.2.1.1.1.2.7.100.101.102.97.117.108.116 = INTEGER: 1
...
```
```text
~/.snmp/mibs$ snmpwalk -v2c -c public -Ln -On 192.168.0.3 hwDhcpRelayMib
.1.3.6.1.4.1.2011.5.7.1.1.3.0 = INTEGER: 1
.1.3.6.1.4.1.2011.5.7.1.1.4.0 = INTEGER: 0
.1.3.6.1.4.1.2011.5.7.1.1.5.0 = INTEGER: 0
...
```
```text
~/.snmp/mibs$ snmpwalk -v2c -c public -Ln -Of 192.168.0.3 hwDhcpRelayMib
.iso.org.dod.internet.private.enterprises.huawei.huaweiMgmt.hwDhcp.hwDHCPRelayMib.1.3.0 = INTEGER: 1
.iso.org.dod.internet.private.enterprises.huawei.huaweiMgmt.hwDhcp.hwDHCPRelayMib.1.4.0 = INTEGER: 0
.iso.org.dod.internet.private.enterprises.huawei.huaweiMgmt.hwDhcp.hwDHCPRelayMib.1.5.0 = INTEGER: 0
...
```
```text
~/.snmp/mibs$ snmpget -v2c -c public -Ln -On 192.168.0.3 .1.3.6.1.4.1.2011.5.7.1.1.3.0
.1.3.6.1.4.1.2011.5.7.1.1.3.0 = INTEGER: 1
```
```text
~/.snmp/mibs$ snmpget -v2c -c public -Ln -Of 192.168.0.3 .1.3.6.1.4.1.2011.5.7.1.1.3.0
.iso.org.dod.internet.private.enterprises.huawei.huaweiMgmt.hwDhcp.hwDHCPRelayMib.1.3.0 = INTEGER: 1
```
```text
~/.snmp/mibs$ snmpwalk -v2c -c public 192.168.0.3 HUAWEI-MIB::hwDatacomm.42.2.1.16.1.3.0
HUAWEI-MIB::hwDatacomm.42.2.1.16.1.3.0 = INTEGER: 16384
```
```text
~/.snmp/mibs$ snmpget -v2c -c public 192.168.0.3 HUAWEI-MIB::hwDatacomm.42.2.1.16.1.3.0
HUAWEI-MIB::hwDatacomm.42.2.1.16.1.3.0 = INTEGER: 16384
```
```text
~/.snmp/mibs$ snmpget -v2c -c public -On 192.168.0.3 HUAWEI-MIB::hwDatacomm.42.2.1.16.1.3.0
.1.3.6.1.4.1.2011.5.25.42.2.1.16.1.3.0 = INTEGER: 16384
```
```text
~/.snmp/mibs$ snmpget -v2c -c public 192.168.0.3 .1.3.6.1.4.1.2011.5.25.42.2.1.16.1.3.0
HUAWEI-MIB::hwDatacomm.42.2.1.16.1.3.0 = INTEGER: 16384
```


## Tips
The `_` symbol is not supported by snmp utilities. You can solve it like this:
- `sed -i 's/_/-/' *`
- use the `-Pu` flag (allow underlines in the MIB)
