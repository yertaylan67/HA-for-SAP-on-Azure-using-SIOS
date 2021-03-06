# Azure Virtual Machines high availability for SAP NetWeaver on Linux using SIOS Protection Suite

 This article describes how to deploy the virtual machines, configure the virtual machines, install the cluster framework, and install a highly available SAP NetWeaver 7.50 system, using SIOS Protection Suite. In the example configurations, installation commands etc., the ASCS instance is number 00, the ERS instance number 10, the Primary Application instance (PAS)  and the Application instance (AAS) is 00. SAP System ID S4D is used.

 This article explains how to achieve high availability for SAP NetWeaver application with the SIOS Protection Suite for RHEL & SLES. The database layer is covered in detail in this [article](HA-for-SAP-HANA-DB.md).

 Read the following SAP Notes and papers first

 SAP Note [1662610](https://launchpad.support.sap.com/#/notes/1662610) Support details for SIOS Protection Suite for Linux

 SAP Note [1928533](https://launchpad.support.sap.com/#/notes/1928533), which has:

- The List of Azure VM sizes that are supported for the deployment of SAP software.
- Important capacity information for Azure VM sizes.
- Supported SAP software, and operating system (OS) and database combinations.
- Required SAP kernel version for Windows and Linux on Microsoft Azure.

 SAP Note [2015553](https://launchpad.support.sap.com/#/notes/2015553) lists prerequisites for SAP-supported SAP software deployments in Azure.

 SAP Note [2002167](https://launchpad.support.sap.com/#/notes/2002167) has recommended OS settings for Red Hat Enterprise Linux

 SAP Note [1984787](https://launchpad.support.sap.com/#/notes/1984787) has general information about SUSE Linux Enterprise Server 12.

 SAP Note [2178632](https://launchpad.support.sap.com/#/notes/2178632) has detailed information about all monitoring metrics reported for SAP in Azure.

 SAP Note [2191498](https://launchpad.support.sap.com/#/notes/2191498) has the required SAP Host Agent version for Linux in Azure.

 SAP Note [2243692](https://launchpad.support.sap.com/#/notes/2243692) has information about SAP licensing on Linux in Azure.

 SAP Note [1999351](https://launchpad.support.sap.com/#/notes/1999351) has additional troubleshooting information for the Azure Enhanced Monitoring Extension for SAP.

 [SAP Community WIKI](https://wiki.scn.sap.com/wiki/display/HOME/SAPonLinuxNotes) has all required SAP Notes for Linux.

 [Azure Virtual Machines planning and implementation for SAP on Linux](https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/sap/planning-guide)

 [Azure Virtual Machines deployment for SAP on Linux](https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/sap/deployment-guide)

 [Azure Virtual Machines DBMS deployment for SAP on Linux](https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/sap/dbms-guide)

## 1. Overview

High availability(HA) for SAP Netweaver central services requires shared storage. To achieve that on Linux so far it was necessary to build separate highly available NFS cluster.
This document describes how to achieve High Availability for SAP, using the SIOS Protection Suite for a Linux VM. SIOS provides High Availability for SAP (A)SCS shared storage. SIOS Datakeeper is used to replicate the volumes/disks between cluster nodes. The SIOS Protection Suite can also used with Azure NetApp Files which eleminates the need for SIOS Datakeeper.

![ASCS](/SAP-ASCS/Images/ASCS-cluster-avset.png)  

![SIOS-Components](/SAP-ASCS/Images/SIOS-Components-Functions-1.png)

The NFS server, SAP NetWeaver ASCS, SAP NetWeaver SCS and the SAP HANA database use virtual hostname and virtual IP addresses. On Azure, a load balancer is required to use a virtual IP address. The following list shows the configuration of the (A)SCS and ERS load balancer.

(A)SCS

Frontend configuration

- IP address 11.1.2.60

Backend configuration

- Connected to primary network interfaces of all virtual machines that should be part of the (A)SCS/ERS cluster

Probe Port

- Port 36<nr>

Load balancing rules

- 32<nr> TCP
- 36<nr> TCP
- 39<nr> TCP
- 81<nr> TCP
- 5<nr>13 TCP
- 5<nr>14 TCP
- 5<nr>16 TCP

## 2. Provission SAP (A)SCS, ERS and Witness Infrastructure

Use the Terraform script from [Proving Ground Infrastructure Provisioning Git](https://github.com/BalaAnbalagan/SAP-on-Azure-Proving-Ground) to deploy all required Azure resources, including the virtual machines, availability set etc., in this example we are not using Load Balancer. You can also deploy the resources manually.

Please follow the respective document in the Proving Ground Infrastructure Provisioning Git

## 3. Install Azure CLI  

 Install Azure CLI on the (A)SCS cluster nodes which is a pre-requisites for SIOS Enhanced Azure IP GenApp. Please refer the installation procedure of the respective to OS

- [RHEL](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-yum?view=azure-cli-latest)  
- [SLES](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-zypper?view=azure-cli-latest)  

 Please login to portal.azure.com from the server.

 ```bash
 az login --use-device-code
 ```

 ```console
 To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code B3D42JUFD to authenticate
 ```

## 4. Install SIOS Protection Suite & Recovery Kits



The following SIOS components are installed in respective nodes.

LifeKeeper Core

- azsuascs1
- azsuascs2
- azsusapwit1

Witness/Quorum

- azsusapwit1

Note:- recommended to use 1 witness/cluster

DataKeeper, SAP Application Reovery Kit & IP Recovery Kit

- azsuascs1
- azsuascs2

[Please follow the installation screenshots here](/SIOS/Install-SPS-Components.md)

## 5. Create Communication Path between Cluster Nodes and Witness

To create the communication path between a pair of servers, you must define the path individually on both servers. LifeKeeper allows you to create both TCP (TCP/IP) and TTY communication paths between a pair of servers. Only one TTY path can be created between a given pair. However, you can create multiple TCP communication paths between a pair of servers by specifying the local and remote addresses that are to be the end-points of the path. A priority value is used to tell LifeKeeper the order in which TCP paths to a given remote server should be used.

Please refer the screenshots on [how to create communication path](Create-Comm-path-SCS.md)

## 6. Create Floating IP for (A)SCS

In this section we will be using the SIOS Enhanced Azure IP Generic Application which creates the secondary IP Configuration for the given NIC on the VM

 Azure IP GenApp run the following Azure CLI command to Switch the Secondary IP from one node to the other in a cluster.

 ```bash
 az network nic ip-config create --resource-group SIOS-SUSE --nic-name NIC_APP-azsuascs1 --private-ip-address 11.1.2.60 --name S4DASCS
 ```

  Note:

- SIOS Enhanced IP GenApp adds 2 mins to the failover time

- It can be used in scenario's where ILB is not avaialble

- While using Azure ILB, this step is not required

The SIOS IP Recovery Kit is used to failover the IP between the cluster nodes.

Please refer the following links to create the resources

- ### [1. Create SIOS Enhanced Azure IP Gen App Resource for (A)SCS](Create-Azure-IP-GenApp-scs.md)

- ### [2. Create IP Resource for (A)SCS](Create-IP-Resource-scs.md)

## 7. Install SAP NetWeaver (A)SCS in Node-1

 Install SAP NetWeaver ASCS as root on the first node using a virtual hostname that maps to the IP resouce created in previous step i.e.,  s4dascs, 11.1.2.60  

 You can use the sapinst parameter SAPINST_REMOTE_ACCESS_USER to allow a non-root user to connect to sapinst. You can use parameter SAPINST_USE_HOSTNAME to install SAP, using virtual hostname.

 ```bash
 sudo <swpm>/sapinst SAPINST_REMOTE_ACCESS_USER=sapadmin SAPINST_USE_HOSTNAME=S4DASCS
 ```

[Please refer the SAP Installation Screenshots](SAPINST-ASCS-NODE1.md)

```bash
/usr/sap/S4D/ASCS00/exe/sapcontrol -prot NI_HTTP -nr 00 -function GetProcessList
```

```console
01.05.2019 12:42:17
GetProcessList
OK
name, description, dispstatus, textstatus, starttime, elapsedtime, pid
msg_server, MessageServer, GREEN, Running, 2019 05 01 12:37:23, 0:04:54, 104629
enserver, EnqueueServer, GREEN, Running, 2019 05 01 12:37:23, 0:04:54, 104630
sapwebdisp, Web Dispatcher, GREEN, Running, 2019 05 01 12:37:23, 0:04:54, 104631
gwrd, Gateway, GREEN, Running, 2019 05 01 12:37:23, 0:04:54, 104632
```

Please update & change the following (A)SCS Instance Profile Parameter

- Autostart = 1 --> 0
- Restart_Program_00 = local $(_ER) pf=$(_PFL) NR=$(SCSID) --> Start_Program_00 = local $(_ER) pf=$(_PFL) NR=$(SCSID)

Note:

Please take backup of the profile before proceeding with installing on the seconday node.

## 8. Install SAP NetWeaver ERS on Node-1

 Install SAP NetWeaver ERS as root on the First node using a physical hostname and the instance number is 10.

 You can use the sapinst parameter SAPINST_REMOTE_ACCESS_USER to allow a non-root user to connect to sapinst.

 ```bash
 sudo <swpm>/sapinst SAPINST_REMOTE_ACCESS_USER=sapadmin
 ```

[Please refer the SAP Installation Screenshots](SAPINST-ERS-NODE1.md)

```bash
/usr/sap/S4D/ERS10/exe/sapcontrol -prot NI_HTTP -nr 10 -function GetProcessList
```

```console
1.05.2019 15:00:08
GetProcessList
OK
name, description, dispstatus, textstatus, starttime, elapsedtime, pid
enrepserver, EnqueueReplicator, GREEN, Running, 2019 05 01 14:58:12, 0:01:56, 18981
```

## 9. Create Data Replication Resource for (A)SCS Mount Point

In this section we are using SIOS DataKeeper to replicate the mount point /usr/sap/S4D/ASCS00 between the cluster nodes.

SIOS DataKeeper for Linux provides an integrated data mirroring capability for LifeKeeper environments.  This feature enables LifeKeeper resources to operate in shared and non-shared storage environments.

![ ](/99_images/Datakeeper-1.png)

Key Features:

- Host-based data replication leveraging existing LAN/WAN
- Multi-target Replication
- Automatic reversal of source/target during failover
- Block-Level, Volume/LUN replication
- Change-only replication
- Very low overhead
- Modes: Synchronous or Asynchronous
- Prevents full re-syncs via bitmap file

Please refer to the screenshots on [how to configure data replication](create-data-rep-ascs00.md)

Note:
Use SIOS DataKeeper only when shared storage solution for (A)SCS mounts are not available.
Asynchronous Data Replication is supported only for Disaster Recovery Scenarios

## 10. Switch VIP to Node-2

To perform SAP (A)SCS installation on the secondary node we need the floating ip to point to the seconday node. so please fail over the IP resource to the secondary node.

Please refer to the screenshots on [how to failover IP resource to secondary node](Switch-VIP-Node-2.md)

## 11. Install SAP NetWeaver (A)SCS in Node-2

 Install SAP NetWeaver ASCS as root on the first node using a virtual hostname that maps to the IP resouce created in privious step i.e.,  s4dascs, 11.1.2.60  

 You can use the sapinst parameter SAPINST_REMOTE_ACCESS_USER to allow a non-root user to connect to sapinst. You can use parameter SAPINST_USE_HOSTNAME to install SAP, using virtual hostname.

 ```bash
 sudo <swpm>/sapinst SAPINST_REMOTE_ACCESS_USER=sapadmin SAPINST_USE_HOSTNAME=S4DASCS
 ```

```bash
 df -h |grep sap
```

```console
Filesystem                                         Size  Used Avail Use% Mounted on
/dev/sdc1                                           64G  240M   64G   1% /usr/sap
/dev/sdf1                                           64G  362M   64G   1% /usr/sap/S4D
/dev/sde1                                           64G  259M   64G   1% /usr/sap/S4D/ERS10
pg-nfs01.provingground.net:/export/media           4.0T  149G  3.9T   4% /sapmedia
pg-nfs01.provingground.net:/export/media/sapmnt    4.0T  149G  3.9T   4% /sapmnt
pg-nfs01.provingground.net:/export/media/saptrans  4.0T  149G  3.9T   4% /usr/sap/trans
```

NOTE: Make sure the disk being replicated is NOT mounted here.

During this SAP (A)SCSinstallation installer, we will be using /usr/sap/S4D mount for ASCS and this installation is required to get the Installation directories, User Environment created for (A)SCS.

[Please refer the screenshots on [how to install SAP (A)SCS Installation](SAPINST-ASCS-NODE1.md)

```bash
/usr/sap/S4D/ASCS00/exe/sapcontrol -prot NI_HTTP -nr 00 -function GetProcessList
```

```console
01.05.2019 12:42:17
GetProcessList
OK
name, description, dispstatus, textstatus, starttime, elapsedtime, pid
msg_server, MessageServer, GREEN, Running, 2019 05 01 12:37:23, 0:04:54, 104629
enserver, EnqueueServer, GREEN, Running, 2019 05 01 12:37:23, 0:04:54, 104630
sapwebdisp, Web Dispatcher, GREEN, Running, 2019 05 01 12:37:23, 0:04:54, 104631
gwrd, Gateway, GREEN, Running, 2019 05 01 12:37:23, 0:04:54, 104632
```

## 12. Install SAP NetWeaver ERS on Node-2

 Install the SAP NetWeaver ERS as root on the Node-2 using a physical hostname and the instance number is 10.

 You can use the sapinst parameter SAPINST_REMOTE_ACCESS_USER to allow a non-root user to connect to sapinst. You can use parameter SAPINST_USE_HOSTNAME to install SAP, using  virtual hostname.

 ```bash
 sudo <swpm>/sapinst SAPINST_REMOTE_ACCESS_USER=sapadmin
 ```

During this SAP (A)SCSinstallation installer, we will be using /usr/sap/S4D mount for ASCS and this installation is required to get the Installation directories, User Environment created for ERS.

[Please refer the Node-1 SAP Installation Screenshots](SAPINST-ERS-NODE1.md)

```bash
/usr/sap/S4D/ERS10/exe/sapcontrol -prot NI_HTTP -nr 10 -function GetProcessList
```

```console
1.05.2019 15:00:08
GetProcessList
OK
name, description, dispstatus, textstatus, starttime, elapsedtime, pid
enrepserver, EnqueueReplicator, GREEN, Running, 2019 05 01 14:58:12, 0:01:56, 18981
```

## 13. Switch Back VIP to Node-1

Switch back the VIP resource back to node to proceed with the SAP cluster configuration.

Please find the screenshots on [how to failback VIP to node-1](Switch-VIP-Node-1.md)

## 14. Create SAP Resource for (A)SCS

Pleas find the screenshots on [how to create SAP (A)SCS Resource](Create-sap-ascs00.md)

## 15. Create SAP Resource for ERS

Pleas find the screenshots on [how to create SAP ERS Resource](Create-sap-ers10.md)

![SIOS SAP (A)SCS/ERS Cluster](/99_images/create-sap-res-s4d-ers10-22.png)

## 16. Install database Instance

  In this example, SAP NetWeaver is installed on SAP HANA. You can use every supported database for this installation. For more information on how to install SAP HANA in Azure, see High availability of SAP HANA on Azure VMs on Red Hat Enterprise Linux. For a list of supported databases, see SAP Note [1928533](https://launchpad.support.sap.com/#/notes/1928533).

  Run the SAP database instance installation

  Install the SAP NetWeaver database instance as root using a virtual hostname that maps to the IP address of the floating hostname of the database for example s4ddb and 11.1.2.50.

  You can use the sapinst parameter SAPINST_REMOTE_ACCESS_USER to allow a non-root user to connect to sapinst.

  ```bash
  sudo <swpm>/sapinst SAPINST_REMOTE_ACCESS_USER=sapadmin
  ```

## 17. SAP NetWeaver application server installation

  Follow these steps to install an SAP application server.

  Prepare application server.

  Follow the steps in the chapter SAP NetWeaver application server preparation above to prepare the application server.

- Install SAP NetWeaver application server.

- Install a primary or additional SAP NetWeaver applications server.

  You can use the sapinst parameter SAPINST_REMOTE_ACCESS_USER to allow a non-root user to connect to sapinst.

  ```bash
  sudo <swpm>/sapinst SAPINST_REMOTE_ACCESS_USER=sapadmin
  ```

## 18. Update SAP HANA secure store

  Update the SAP HANA secure store to point to the virtual name of the SAP HANA System Replication setup.

  Run the following command to list the entries as \<sid>adm

  ```bash
  hdbuserstore List
  ```

  This should list all entries and should look similar to

  ```console
  DATA FILE : /home/s4dadm/.hdb/azsusap1/SSFS_HDB.DAT
  KEY FILE : /home/s4dadm/.hdb/azsusap1/SSFS_HDB.KEY
  KEY DEFAULT
  ENV : s4ddb.provingground.net:30013
  USER: S4HABAP
  DATABASE: S4D
  ```

## 19. Failover Testing

Right click on the Resource to be failed over to other node and select "in service".

The process takes close to 4 minutes and 2 minutes spend on failing over the Azure IP.

Please refer the screenshots on [testing (A)SCS fail-over testing](SIOS-ascs-Failover-Testing.md)
