# **Nimble Storage Array Replication** with **Witness** and **Peer Persistence** 

## DISCLAIMER
1 **THIS IS NOT A FULLY QUALIFIED GUIDE**

> Please refer to HPE - Nimble guidance document or reach out to HPE representatives for advanced replication setup of High-Availability on Nimble Storage Array

2 Nimble storage array used in this document is Model HF20
> The topology and configuration used in this document might have been changed and not documented properly

## TOPOLOGY
![topology](https://user-images.githubusercontent.com/63829388/199404103-d9905cec-455d-49aa-a610-8150817d7eb6.jpg)

> green line shows data links, while red line shows management link

## REQUIREMENTS and PRE-REQUISITES
- Models from same class
  >Peer Persistence can be configured between same models. Currently it is not possible to sync replicate a HF20 to a HF40 for example, even though the latter one is more powerful and could take the additional load.
- **Replication works over IP**
  > Sync replication traffic ***can be carried over IP only, there is no option to use fibre channel***. This should not be considered as an issue, at least regarding the count of the 10Gbit-T connections as all models have at least two onboard. Worth to remember, front end connectivity – so array to host – media can be FC.
- Arrays in PP relation should use same protocol, so either fibre channel or iSCSI. ***Mixed protocol in one PP group is not allowed***.
- Group, replication and management traffic must use the same L2 – **not one L2 segment all together** – segment, so ports used for a certain traffic should belong to the same L2 segment.
- **Maximum 5ms round trip time between the two arrays.**
- ***(optional)***
  > - Installation of witness, located on a third site. This is not optional if Peer Persistence is the goal to set up, but optional if manual activation of volumes required upon failover on the surviving array.

## CONFIGURATION
Steps to be taken

1. Upgrade both arrays to Nimble OS 5.1. No array can have it currently by default over Infosight, but it can be requested from Nimble Support to whitelist the array. Only thing they ask is the serial of the storage and soon it will appear.

2. Add both array to one group. I will not describe this, as this is not really new. It works the same way if you have at least two Nimble boxes. This way the arrays will become one as management entity, one will be acting as group leader and will host the configuration and the management interface.
   > ![Nimble group from GUI](https://user-images.githubusercontent.com/63829388/199404386-da2c5b19-f182-421a-9bce-09046b058917.png)

   > ![Nimble group from CLI](https://user-images.githubusercontent.com/63829388/199404378-949d8be4-e87c-421f-8cd6-e29b4717a334.png)

3. (only applicable if iSCSI is used as frontend protocol) Both storage should be set to use “Group Scoped Target” instead of “Volume Scoped Target”. Table below shows why:
   > ![Table configuration for iSCSI](https://user-images.githubusercontent.com/63829388/199405740-0bfd0e89-1080-4d9d-bc55-cd62fb911b65.png)

   > [Nimble guidance can be found here](https://infosight.hpe.com/org/8f74fd94-28dd-4d88-a5b7-7133fa764297/resources/nimble/docs)

   > [Nimble Configuration matrix validation can be found here](https://infosight.hpe.com/org/8f74fd94-28dd-4d88-a5b7-7133fa764297/resources/nimble/validated-configuration-matrix)

    It is required because a volume export contains the array’s IQN as target, but since the volume itself can move between two arrays that would lead to unavailability so it should use the group name in the IQN instead.

    > ![Nimble iSCSI group setup](https://user-images.githubusercontent.com/63829388/199406098-7fcd43bf-b4ac-4b09-8cdc-d72502a7cb27.png)

4. Nimble Connection Manager deployment. This is important since this will set the Path Selection Policy in VMware to NIMBLE_PSP_DIRECTED on given LUNs.

    ![Path selection policy](https://user-images.githubusercontent.com/63829388/199406259-4f0d3e8f-268f-4647-81ea-72715e66d6d6.png)

5. Witness installation. As mentioned before, this is optional and required only if we ant ASO (Automatic SwitchOver), which is automatically brings the Peer Persistence synchronously replicated volume to “active” state on the surviving array. If manual failover is enough this can be skipped. Currently witness is an RPM file, which can be downloaded from HPE Infosight and can be installed on a CentOS machine (minimum 7.2). Later there will be an appliance for this, but not yet public. Important to open up firewall – if there is any – as arrays’ management IP should reach it over port 5395.
   
6. Set Witness

    Setup witness via GUI

    ![Setup witness via GUI](https://user-images.githubusercontent.com/63829388/199406553-1804d43a-ef44-440c-8602-1c564ca4472d.png)

    or via CLI

    ![Setup witness via CLI](https://user-images.githubusercontent.com/63829388/199406559-e1351134-bdcd-4c76-a5c4-05cbc46c7741.png)

7. Creation of necessary Volume Collections. One is enough, but in that case only those volumes will be replicated which are member of that particular volume collection, so direction will be source-to-destination. In case of failover and after the recovery, the direction will be reversed, but still run in one direction. If we want to have volumes that are active on HF01 and some on array HF02, we need at least two groups, since the replication direction is defined on volume group level.
   
    Create HF01 to HF02
    ![Create volume collection](https://user-images.githubusercontent.com/63829388/199406761-c63a21da-bea1-400f-9ca8-2d524367ca5e.png)

    It can be seen that even though i selected “No protection template” it will still set the snapshot retention to two. It can be modified to one, but it will show an error if “Do not replicate” is selected. I am not totally clear at the moment why is this, but will find out.

    ![Volueme Collections](https://user-images.githubusercontent.com/63829388/199406765-0fff7c02-45a6-4859-a7a4-ebc6347ab478.png)

    It is more important to set the “Replication partner” properly. Above the replication partner is HF02 for volume collection “NimbleHF01-to-HF02”. I’ve created a second volume collection, named “NimbleHF02-to-HF01” in which the partner is HF01. This can be seen here:

    ![Volume Collections](https://user-images.githubusercontent.com/63829388/199406767-f3411e9c-c396-4eaa-8fec-70351d903cfe.png)

8. Almost seeing the chequered flag, create some volumes according to this table:
   
    |      LABEL          |   POOL  | LUN ID  |  SIZE  |
    |---------------------|---------|---------|--------|
    |Nimble-PP-LUN01-HF01 |HF01-Pool| 01      | 500 GB |
    |Nimble-PP-LUN02-HF02 |HF02-Pool| 02      | 500 GB |
    |Nimble-LUN03-HF01    |HF01-Pool| 03      | 500 GB |
    |Nimble-LUN04-HF01    |HF02-Pool| 04      | 500 GB |

    First two volumes will be replicated the third will be only local on HF01, the last local on HF02 array.

    ![Create new volume](https://user-images.githubusercontent.com/63829388/199406772-b52c1d83-5c45-4022-accc-966348cfa581.png)

    After pressing “Create” button, the volume will appear twice, one will be “upstream”, the other will be “downstream”. No matter how they name this, it is active and standby.

    ![Create new volume](https://user-images.githubusercontent.com/63829388/199406773-7e74a6ea-f8d1-4c31-a976-4736eecf0a19.png)

    Let’s create **“Nimble-PP-LUN02-HF02”** volume:

    ![Create new volume](https://user-images.githubusercontent.com/63829388/199406777-e9e5b62b-2262-4287-96d5-54e1d886fa8d.png)

    Finally the “local” volume creation:

    ![Create new volume](https://user-images.githubusercontent.com/63829388/199406780-75453ec8-8b78-49d2-a1ee-dbeadc3fd5e3.png)

    If all successfully done we will get this window and state below:

    ![Create new volume](https://user-images.githubusercontent.com/63829388/199406752-3e1a567c-de4e-4e68-a85a-13eae5659214.png)

9.  I am using VMware, so just need to format the volumes to VMFS – I will name them in the same way as the volumes:
    
    ![Create new datastore](https://user-images.githubusercontent.com/63829388/199406758-a9968d9f-62f6-4b0f-acfa-772eee3b09f3.png)

    If everything is by the book the end result in VMware will look like this – regarding the first four datastore your mileage might vary:

    ![Datastores](https://user-images.githubusercontent.com/63829388/199406759-71de0829-c4cd-4db1-8b82-c28a8f1cc769.png)

    We are at the point where and when Nimble Peer Persistence setup is completed.

## TESTING
In the picture about the test architecture you can see two virtual machines, named VM1, located on datastore “Nimble-PP-LUN02-HF01” and VM2, located on datastore “Nimble-PP-LUN02-HF02”. I have two hosts in the EPYC cluster, VM1 runs on host1, VM2 runs on host2. I have started an IOmeter on both VMs and set four workers and use the following profile (16k, 50% random, 70-30 read-write):

![Test setup](https://user-images.githubusercontent.com/63829388/199407524-c8b98ded-0c38-484f-b04c-f093e19df650.png)

This is not a performance benchmark, I just want to generate some activity to have traffic between the two arrays. The load sets to a baseline:

![Load benchmark](https://user-images.githubusercontent.com/63829388/199407527-024bdae7-9463-4453-93f2-c4e43ee922b8.jpg)

Same “load” viewing through the Nimbe management interface:

![Load test](https://user-images.githubusercontent.com/63829388/199407530-a3867622-c0f1-4bb4-be75-90eef9983dc4.png)

After each test round I will wait till the successful reconvergence and if upstream array is changed I am setting it back to the original array.

### **First test**: Management connection interruption

An array is losing the connectivity to the management network, so both of it’s controllers will have the mgmt port offline.

> Result: no failover, no outage. The management interface puts up a warning showing no connection, but array is still manageable.
    
> ![First test](https://user-images.githubusercontent.com/63829388/199407533-bc6f6fbf-b434-4e18-8b76-71267342ed7c.png)

> ![First test](https://user-images.githubusercontent.com/63829388/199407535-55f60f96-a36c-4e76-869e-56014ef7d7f4.png)

### **Second test**: Active controller failure in array HF02

Immediate failover to the standby controller in the same array, so in HF02.

> Since this is local failover, it has some impact on virtual machines – in this case VM2 – since the IO will be on “hold” during the activation of the standby controller and process to activate the volume. SCSI timeout is 30 seconds, and this hold time might vary, but usually takes 10-20 seconds.

> ![Second test](https://user-images.githubusercontent.com/63829388/199407536-2c24c263-9ee8-4b23-8f6e-fa085b9728ab.png)

> Peer replication is still healthy:

> ![Second test](https://user-images.githubusercontent.com/63829388/199407539-876efd05-1ee1-40e7-8d85-47ffb4b8674f.png)

### **Third test**: I will remove power cords from HF2 array power supplies.

Response it total failover between arrays. So replicated and peer persistence protected volumes will be placed in “upstream” state on surviving array by using witness as tie breaker. Non replicated local volumes will be offline, at least the ones – in my case “Nimble-LUN03-HF02” – which are local on one array.

> ![Third test](https://user-images.githubusercontent.com/63829388/199407543-1fa3315e-5db8-4f53-9d04-a3d2d58161cf.png)

> Replication state surfaces up the error as well:

> ![Third test](https://user-images.githubusercontent.com/63829388/199407546-5e1effce-fce4-43d4-a35b-2be7cbe5da79.png)

> From VMware the state is shown below in the form of Dead paths:

> ![Third test](https://user-images.githubusercontent.com/63829388/199407547-fe1bd9b7-079e-4e43-ba7d-1f6fbc0ee396.png)

> Virtual machinces located on datastores that are peer persistence protected replicated volumes will have their IO on hold for 10-20 seconds – just like in test two – so VM2 in this case will need to hold breath a little, up until the point when HF01 brings datastore Nimble-PP-LUN02-HF02 up in upstream state.

### Further what-if tests
What happens if I remove two capacity drives and put them into the other’s bay. Well, nothing as you can see below – don’t search the fault
![Couriusity test](https://user-images.githubusercontent.com/63829388/199407552-9ee9175a-db60-4f3b-ad50-479dd8d9a786.png)

What happens if I pull out two SSD from the array – since this is a hybrid array I am removing cache – affecting write/read cache operations? Nothing, besides receiving alerts and the decreased FDR – flash-to-disk ratio – and capacity. Note the top right capacity in the picture which is reduced since removed two 480GB cache drives.

![Couriusity test](https://user-images.githubusercontent.com/63829388/199407519-eb508923-2267-4cc6-90d7-a7233dac9c74.png)
> **What if I remove all capacity disks, shuffle and put them back in some totally different order. Nothing. I have no words to describe the surprise on my face**.

## OTHER THINGS TO REMEMBER
Sync replicated volumes are synchronized to the other array – so from source to target – in non deduplicated and/or compressed format, as that would require additional CPU cycles to process at source side. So it is better to replicate it immediately when hitting the source array active controller NVDIMM module and let both arrays take the load to deduplicate/compress them. Latency is more important here than the amount of replication traffic.

Also important that sync replicated volumes cannot be resized, so choose wisely or later you might be looking at breaking the replication, do the resize and add it back.

## SUMMARY
Solution works well, I have tested failover around 20 times and there was only a single occurrence when automatic failover was not working. Remember, Nimble OS 5.1.1 is still not in production, so they will further improve the code and once released it will pass all tests. HPE Nimble is now advancing to an area where earlier some more expensive and certainly more complicated storage systems are/were.
