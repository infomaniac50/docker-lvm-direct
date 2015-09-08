# Docker Direct LVM on ArchLinux

sudo is not used here because ArchLinux does not have it by default.
You will need to have superuser privileges or assign your user to the docker group.

## Before we Start

### Saving your Data

If you have anything in docker you want to keep, you must export it or have some Dockerfiles ready to go when all is said and done.

**WARNING!** This process **WILL** destroy all containers, images and metadata just as if docker was just installed.

### The Shutdown
Before we do anything we have to stop the docker daemon
```bash
systemctl stop docker
```

### The Real Stuff

Next create a new partition or use an existing block device.

I assume if you have come this far with ArchLinux, you also know how to create a simple disk partition. That said, Gparted from the Gnome project can still be a big help here. However, if you're going to use a whole disk then skip this step and omit the partition number from the physical volume name.

Now we can proceed to creating the LVM.

## Creating LVM Storage for Docker

### Some Variables to Reference Later

#### Thin Pool Name
This is the name of the thin pool device docker will use. Naming isn't too important but use alphanumeric characters only. During one attempt I got a naming error because of a hyphen. The general idea is to keep the names simple and organized. One of the articles I read used pool0, but I decided to use something a little more descriptive.

```bash
thinpoolname=docker
```

#### Block Storage
You need to replace this variable with the raw partition or block device name. Mine was /dev/sdb3. eg. second disk, third partition.

NOTE: My second disk uses GPT partitioning so I can have as many primary partitions as I want, while MBR is limited to 4 primary partitions.

```bash
blockstorge=<Your raw device or partition goes here>
```
### Creating the LVM
#### Create a Physical Volume

```bash
pvcreate $blockstorage
```

This one is fairly easy and provides the *free space* for use with the volume group which we'll create in the next step. Simply enter the device name to use as physical block storage.

#### Create a Volume Group
Next add the physical volume to a new volume group.

This step pools physical volumes into one giant storage device. It's sort of like adding a new disk to a RAID array or btrfs filesystem. It can be any block device as long as it's available when the volume group gets loaded. Examples are a SCSI LUN number, RAID array, NFS mount, Internal Hard Drive or SSD, etc.

```bash
vgcreate $thinpoolname $blockstorage
```

#### Create the Logical Partitions

On my first/third try at doing this I made the logical volumes too big. I got an error about how the volume group was not big enough for the thin-pool.

What the frak, man! Why does crap never work the way you expect! I nearly gave up at this point, but I doubled down and tried again. Needless to say this whole process took at least five tries to get right but it payed off in the end.

On that note, you should have a nice large margin between the end of the (data + metadata), and the end of the volume group. If you only have one physical volume in the volume group then the thin-pool conversion will be limited to the size of that device.

I believe this is because the logical volume starts out small and then grows to fill the physical volumes of the volume group. You can actually over-provision the logical volumes with more space than is actually available. Then when the volume group starts to outgrow your actual capacity you just add more block storage.

NOTE: I'm a little OCD so I like even numbers. Since gparted uses powers of 2 I chose a nice middle ground for my 6TB WD Red. I have other stuff in here so 1 Terabyte seemed a little too greedy. 256 Gigabytes seemed too Windows Vista sized.

What! 2006 is where it's at dude!

My first physical volume was 512 Gigabytes (2^19 Megabytes or 549,755,813,888 bytes).

```bash
lvcreate -n $thinpoolname-data -L 10G $thinpoolname
lvcreate -n $thinpoolname-meta -L 1G $thinpoolname
```

#### Convert the Logical Partitions to a Thin Pool

The next and final step is to convert the data and metadata logical volumes into a combined thin-pool volume.

```bash
lvconvert --type thin-pool --poolmetadata $thinpoolname/$thinpoolname-meta $thinpoolname/$thinpoolname-data
```

#### Checking Your Work
If everything went as planned you should see something like the following when you run ``dmsetup ls``.

```bash
dmsetup ls
# docker-docker--data (254:2)
# docker-docker--data_tdata (254:1)
# docker-docker--data_tmeta (254:0)
```

### Veto Powers Override

#### I Reject Your Config and Substitute My Own

After the LVM is setup, copy the docker systemd service file from /usr to the override file in /etc. The /etc directory gives you a place to put custom systemd config files without editing the pacman controlled /usr files.

```bash
cp /usr/lib/systemd/system/docker.service /etc/systemd/system/docker.service
```

Then edit the file using your favorite editor and follow the gist of etc/systemd/system/docker.service in this repository.

```bash
$EDITOR /etc/systemd/system/docker.service 
```

#### Something New is Around the Corner

The systemd service files have changed so tell systemd to reload its configuration.

```bash
systemctl daemon-reload
```

### The Big Test

Now with all the hard stuff out of the way, go ahead and start the docker daemon.

```bash
systemctl start docker
```

If everything is working properly then docker will create a new graph at /var/lib/docker. You might have to wait a bit for it to finish bootstrapping the directory. Even on my 8 core 12GB machine it takes a while to create all the files and folders in /var/lib/docker.

## Your Done

What? Were you looking for more?

OK, here's some links:

- https://docs.docker.com/reference/commandline/daemon/
- https://docs.docker.com/reference/commandline/daemon/#storage-driver-options
- https://github.com/docker/docker/issues/4639
- https://github.com/docker/docker/tree/master/daemon/graphdriver/devmapper
- https://jpetazzo.github.io/2014/01/29/docker-device-mapper-resize/
- https://www.projectatomic.io/blog/2015/06/notes-on-fedora-centos-and-docker-storage-drivers/

Take a look at these too:

- http://man7.org/linux/man-pages/man7/lvmthin.7.html
- ``man lvmthin``