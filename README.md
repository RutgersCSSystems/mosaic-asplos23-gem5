# Mosaic Gem5 Full System Simulation Instructions


## System Requirements for Gem5 full system simulation
----------------------------------------------------
- A Linux system (tested with Debian distribution)
- 4 or more cores
- At least 20GB of free RAM
- 20-30GB disk for sequential runs and 100GB for parallel runs


## (1.) Setting  Gem5 Mosaic on CloudLab (Skip to 2 if not using CloudLab)
-----------------------------------------------------------------------
You have the option to run Gem5 Mosaic on CloudLab (which we used for
development and experiments). If you do not want to use CloudLab, skip to 2.

## 1.1 Instantiating CloudLab nodes before cloning the repo 
We run the gem5 simulation in CloubLab and use its fast NVMe storage to speed up full system simulation. We recommend users use the following profile to instantiate two CloudLab nodes.

**CloudLab profile:** You could use any CloudLab machine with Intel-based CPUs running Debian kernel version 18.04.
**Recommended Instance Types:** We recommend using m510 nodes in UTAH datacenter that have fast NVMe storage and are generally available.
You could also use a pre-created profile, "2-NVMe-Nodes," which will launch two NVMe-based nodes to run several parallel instances.

## 1.2 Partitioning an SSD and downloading the code.
If you are using CloudLab, the root partitions only has 16GB (for example: m510 instances). 
First, set up the CloudLab node with SSD and download the code on the SSD folder.

If you are using the m510 nodes (or 2-NVMe-Nodes profile), the NVMe SSD with 256GB is in
"/dev/nvme0n1p4"

### 1.3 To partition an ext4 file system, use the following commands
```
sudo mkfs.ext4 /dev/nvme0n1p4 
mkdir ~/ssd
sudo mount /dev/nvme0n1p4  ~/ssd
sudo chown -R $USER ~/ssd
cd ~/ssd
```

### 1.4 Now clone the repo
```
git clone https://github.com/RutgersCSSystems/mosaic-asplos23-gem5
```

## (2.) Compilation
------------------------
All the package installations before compilation use debian distribution and "apt-get"  

### 2.1 Setting the environmental variables
MAKE sure to set the correct OS release by assigning the correct
OS_RELEASE_NAME. In Debian distributions, this can be extracted using "lsb_release -a"
```
export OS_RELEASE_NAME="bionic"
source scripts/setvars.sh $OS_RELEASE_NAME
```

### 2.2 Compile gem5 and linux 4.17 kernel. 

Feel free to use other kernel versions if required.

```
./compile.sh
```
## 2.2 QEMU image setup for gem5 full system simulation
By default, the script creates a QEMU image size of 10GB. If you 
would like a larger image, in the image creation script added 
below (create_qemu_img.sh), change the following line as needed. 
```
qemu-img create $QEMU_IMG_FILE 10g
```
```
./create_qemu_img.sh
```

### 2.3 Mount the QEMU image
A successful mount followed by "df" command will show the mounted 
directory. Some VMs might not have /bin/tcsh. So, we will manually 
copy to the VM disk file.
```
test-scripts/mount_qemu.sh
sudo apt-get update
sudo apt-get install build-essential g++
exit //exit from the VM image root
```
### 2.4 Copying applications and gem5 scripts to VM
To copy all apps to the root folder inside the QEMU image, use the following
command. In addition to the application, we also copy the m5 application
required for letting the gem5 host for starting and stoping the simulation.
```
$BASE/test-scripts/copyapps_qemu.sh
(or)
sudo cp -r apps/* $BASE/mountdir/
```

### 2.5 Setting the password for QEMU VM
- Set the username for the VM to root
- Set the VM image password to the letter "s". This is just a QEMU gem5 VM and will not cause any 
issues. *We will make them configurable soon to avoid using a specific password.*

```
test-scripts/mount_qemu.sh
cd $BASE/mountdir
sudo chroot .
passwd
```
Once the password is set, exit the VM and umount the image

```
exit
cd $BASE
test-scripts/umount_qemu.sh
```
### 2.6 Disabling paranoid mode
```
echo "-1" | sudo tee /proc/sys/kernel/perf_event_paranoid
```

## (3.) Running Simulations
----------------------
We use "test-scripts/prun.sh," a reasonably automated script to run the
simulations for different applications.

In the AE, as evaluated in the paper, we have included the sample applications: 
(1) hello, (2) graph500, (3) xsbench, (4) btree, or (5) gups

We support two workloads: 
(1) "tiny" input, with the smallest possible input to quickly check if everything works 
( < 2 minutes for each application except gups and btree)
(2) "large" input as used in the paper (could take several hours to days)


### 3.1 Running gem5 simulator
```
test-scripts/prun.sh $APPNAME $ASSOCIATIVITY $TOCSIZE $TELNET_PORT $USE_LARGE_INPUT
```
**NOTE: For each WAYS and TOCSIZE configuration, a separate copy of the repo is generated along with the VM image, and the simulation is run from the copied folder. Please be mindful of the available disk and memory size.**

#### Examples:
To run graph500, xsbench, btree, gups sequentially (one after the other) with  2-way associativity with TOC size of 4 and TELNET port at 3160, with tiny inputs
```
test-scripts/prun.sh graph500 2 4 3160 0 
sleep 5
test-scripts/prun.sh xsbench 2 4 3160 0
sleep 5
test-scripts/prun.sh btree 2 4 3160 0
sleep 5 
test-scripts/prun.sh gups 2 4 3160 0
sleep 5
```
To run all applications with large inputs
```
test-scripts/prun.sh graph500 2 4 3160 1
sleep 5
test-scripts/prun.sh xsbench 2 4 3160 1
sleep 5
test-scripts/prun.sh btree 2 4 3160 1
sleep 5
test-scripts/prun.sh gups 2 4 3160 1
```

To run btree, direct-mapped with TOC size of 4 and TELNET port at 3161, and use large inputs
```
test-scripts/prun.sh btree 1 16 3161 1
```

For full associativity, we just specify the number of ways as TLB size. 
For example, if the TLB_SIZE=1024 (default) 
```
test-scripts/prun.sh graph500 1024 16 3162 0
```

**Some notes:**
To run an application inside a VM and begin the simulation, we need to login to
the QEMU VM, run gem5's *m5 exit* inside the VM, followed by running an
application (also inside the VM), and finally run *m5 exit* to signal the gem5
simulator in the host that an application has finished execution and it is time
to stop the simulation.

To avoid manually logging into a VM and running an application, we use telnet
and a specific PORT number.


### 3.2 To run multiple instances simultaneously
To run multiple instances in parallel, the instances could be run as background (interactive) tasks.
Simply copy the block from each markdown file and paste it on the terminal for the long running jobs to begin.
The number of jobs to launch depend on the number of cores, memory size, and available disk size.

```
test-scripts/parallel-graph500.md
test-scripts/parallel-btree.md
test-scripts/parallel-xsbench.md
test-scripts/parallel-gups.md
```

You could also do it manually as shown below (for tiny input):

```
exec test-scripts/prun.sh $APPNAME $ASSOCIATIVITY $TOCSIZE $TELNET_PORT $USE_LARGE_INPUT &
```
For example, the following lines run graph500 varying the associativity, keeping the TOC size constant. 
Please make sure to use different port numbers.
```
exec test-scripts/prun.sh graph500 2 4 3160 0 &
sleep 5
exec test-scripts/prun.sh graph500 4 4 3161 0 &
sleep 5
exec test-scripts/prun.sh graph500 8 4 3162 0 &
sleep 5
exec test-scripts/prun.sh graph500 1024 4 3163 0 &
sleep 5
```

The above commands would generate an output in a format shown below:
```
----------------------------------------------------------
RESULTS for WAYS 2 and TOC LEN 4
----------------------------------------------------------
Vanilla TLB miss rate:0.9288%
Mosaic TLB miss rate:0.6426%
----------------------------------------------------------
RESULTS for WAYS 4 and TOC LEN 4
----------------------------------------------------------
Vanilla TLB miss rate:0.8835%
Mosaic TLB miss rate:0.6312%
----------------------------------------------------------
RESULTS for WAYS 8 and TOC LEN 4
----------------------------------------------------------
Vanilla TLB miss rate:0.9007%
Mosaic TLB miss rate:0.6414%
----------------------------------------------------------
RESULTS for WAYS 1024 and TOC LEN 4
----------------------------------------------------------
Vanilla TLB miss rate:0.6971%
Mosaic TLB miss rate:0.6716%
```

For large inputs (e.g., xsbench)
```
exec test-scripts/prun.sh xsbench 2 4 3160 1 &
sleep 5
exec test-scripts/prun.sh xsbench 4 4 3161 1 &
sleep 5
exec test-scripts/prun.sh xsbench 8 4 3162 1 &
sleep 5
exec test-scripts/prun.sh xsbench 1024 4 3163 1 &
sleep 5
```

### 3.5 Setting TLB size
To change the default TLB size, in prun.sh, change the following:
```
TLB_SIZE=1024 => TLB_SIZE=1536
```



### 3.3 Changing application parameters
The inputs for tiny and large inputs for each application can be changed in the follow python scripts
```
$BASE/test-scripts/gem5_client_tiny.py
$BASE/test-scripts/gem5_client_large.py
```
For example, to change the scale and the number of edges 
of graph500 workloads, modify the following command in tiny or large python scripts
```
commands = [b"/m5 exit\r\n", b"/seq-list -s 4 -e 4\r\n", b"/m5 exit\r\n" ]
```


### 3.4 Forceful termination if required
If you would like to terminate all scripts and gem5 simulation, one could use the following commands
```
PID=`ps axf | grep Ways | grep -v grep | awk '{print $1}'`;kill -9 $PID
PID=`ps axf | grep prun.sh | grep -v grep | awk '{print $1}'`;kill -9 $PID
```

## (4.) Result Generation
-------------------
In full system simulation, for each memory reference, we use a Vanilla TLB and
a parallel Iceberg TLB to collect the TLB miss rate for both Vanilla and
Mosaic.

After an application has finished running for a configuration, the miss rates
are printed as shown below.

```
Sample output:

Vanilla TLB miss rate:1.1046%
Mosaic TLB miss rate:0.7307%
```
## 4.1 Result Files
The result files can be seen in the result folder inside the base gem5-linux
($BASE) folder. The result file path is of the following structure:
```
result/$APP/iceberg/$ASSOCIATIVTY/$TOCSIZE
e.g., result/graph500/iceberg/2way/toc4
```


