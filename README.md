# CVAT on Google Cloud for Stanford

This is a quick guide for setting up CVAT on Google Cloud, created by [RSE Services](https://stanford-rc.github.io/rse-services/).
For this example, we will deploy the [cvat](https://github.com/openvinotoolkit/cvat) software on a Google Cloud instance, and take the following design
choices:

## General Plan

### Storage

Storage will be a disk (1TB) for videos, which can be [expanded if needed](https://cloud.google.com/compute/docs/disks/add-persistent-disk).
For the database, we will use cloud managed (and backed up) sql (actually postgres). We will first, however, use a container
just for early testing.

### Compute

As a generaly strategy, we will use compute engine, a single instance to run continuously (or can be stopped to save funds) to run
cvat. The original development will use docker-compose for container-based development (including the database)
and after this initial test, we will connect to a more production cloud managed SQL database.

## Setup

### Snapshot Schedule for the Disk

Before creating your instance along with the disk (in one operation) you likely want to create a
[snapshot schedule](https://console.cloud.google.com/compute/snapshots?_ga=2.69171477.529843139.1601054786-1672382728.1587327586).
For a 1TB disk, a snapshot of the entire thing will cost approximately $25, which is reasonable
for peace of mind about your data on top of a base estimate of ~$170 for the disk itself. 
For the snapshot schedule, you'll want to make sure to select the same zone as your intended instance,
and you can choose a reasonable frequency (e.g., nightly or weekly, depending on how often you need it).
You should also choose a time that the disk won't be used, most reasonably in the wee hours of the morning 
(10AM UTC is 3:00am Pacific time). Keep in mind that this schedule will be used for the attached
persistent disk with data, and not the boot disk of the image. If you want an additional snapshot
of the boot disk, that should be done separately. The json export wasn't working at the writing
of this guide, so a screenshot of the configuration is shown below:

![img/snapshot.png](img/snapshot.png)

### Compute Engine

After you've created your snapshot schedule, the next step is to create your instance on Google Compute Engine. Generally we want the following:

 - n1-standard-4: with 15GB memory. The reason to choose this over n2-standard-4 (16GB) is because GPUs currently only work with the N1 series, and it's also cheaper.
 - http/https access
 - a boot disk with 100MB
 - enable deletion protection
 - attach a persistent disk for 1TB (1000GB)
 - if creating a custom domain, it's good to reserve an external ip address (non-ephemeral)

The [instance config](instance-config.ms) is provided here if you want to reproduce this exact setup,
either using the gcloud command line tool or programatically. Remember that
the boot disk by default will be deleted with the instance, so be sure to edit the instance
and uncheck this box if you don't want this behavior. Although backups (snapshots) for the disk
are possible, we won't be setting them up because the boot disk won't have valuable data,
just the software.

#### Disk Formatting

Once you've created the instance, you'll want to shell in (either with gcloud or cloud shell, the command
is easy to grab from your Compute Instances page). You'll need to [format and attach](https://cloud.google.com/compute/docs/disks/add-persistent-disk?#formatting) the disk.
Here we can use `lsblk` to see our unformatted disk, the last one:

```bash
$ lsblk
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0     7:0    0  30.3M  1 loop /snap/snapd/9279
loop1     7:1    0  55.3M  1 loop /snap/core18/1885
loop2     7:2    0 126.9M  1 loop /snap/google-cloud-sdk/151
sda       8:0    0   100G  0 disk 
├─sda1    8:1    0  99.9G  0 part /
├─sda14   8:14   0     4M  0 part 
└─sda15   8:15   0   106M  0 part /boot/efi
sdb       8:16   0  1000G  0 disk 
```

The identifier of the disk is `sdb`, we can call this the `DEVICE_ID` We know the boot disk is right above it (`sda`)
because of the size. This is the command to format:

```bash
# sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/DEVICE_ID
sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
```
```
mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done                            
Creating filesystem with 262144000 4k blocks and 65536000 inodes
Filesystem UUID: 29d888e5-97d1-46b3-b7f0-e8ea12c91e59
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
	4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968, 
	102400000, 214990848

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done 
```

Finally, you need to mount it somewhere! This set of commands will create the directory
to mount, mount it, and then enable access for all users.

```bash
sudo mkdir -p /mnt/disks/data
sudo mount -o discard,defaults /dev/sdb /mnt/disks/data
sudo chmod -R a+w /mnt/disks/data
```

And add an entry to fstab so the disk mounts when the instance restarts.

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

Get the uuid of the zonal persistent disk

```bash
sudo blkid /dev/sdb
/dev/sdb: UUID="12345" TYPE="ext4"
```

Add a line to the fstab file as follows

```bash
UUID=UUID_VALUE /mnt/disks/MNT_DIR ext4 discard,defaults,NOFAIL_OPTION 0 2
UUID=12345 /mnt/disks/data ext4 discard,defaults,nofail 0 2
```

You can verify that the disk is mounted by taking a look!

```
$ ls /mnt/disks/data/
lost+found
```

### Installing Dependencies

Once the disk is ready, it's time to install cvat. We are following the instructions in the [quick start](https://github.com/openvinotoolkit/cvat/blob/develop/cvat/apps/documentation/installation.md#quick-installation-guide) guide. This is also how we knew to choose Ubuntu 18.04.
We first install dependencies on the host (you should see the link for the most up-to-date.

```bash
sudo apt-get update
sudo apt-get --no-install-recommends install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg-agent \
  software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
sudo apt-get update
sudo apt-get --no-install-recommends install -y docker-ce docker-ce-cli containerd.io
```

And ensure that we don't need sudo to use docker (you can log out and back in after issuing this command).

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```

After logging in and out, test the Docker install with:

```bash
$ docker run hello-world
```

Install docker-compose, which we use for container orchestration.

```bash
sudo apt-get --no-install-recommends install -y python3-pip python3-setuptools git
sudo python3 -m pip install setuptools docker-compose
```

### Installing CVAT

And finally, let's install cvat. Since we possibly want other users to interact with
the install (sometime in the future) let's install to `/opt`

```bash
sudo mkdir -p /opt/apps
sudo chmod a+w /opt/apps
cd /opt/apps
git clone https://github.com/opencv/cvat
cd cvat
```

We'll create the cloud hosted sql database and update the docker-compose file next.

### Creating Cloud Managed SQL Database

Since we want a production, backed up database, we will create on on Google Cloud.
For our cvat instance, this will mean that we will need to remove the postgres container
from the docker-compose.yml, and then define the following environment variables in our
docker-compose.yml for the cvat instance to access. Since there are quite a few changes,
a [docker-compose.yml](docker-compose.yml) file is provided here.

```yaml
    environment:
      DJANGO_MODWSGI_EXTRA_ARGS: ""
      ALLOWED_HOSTS: '*'
      CVAT_REDIS_HOST: "cvat_redis"
      CVAT_POSTGRES_HOST: "[[GOOGLE CLOUD HOST HERE]]"
      CVAT_POSTGRES_DBNAME: "[[GOOGLE CLOUD DATABASE NAME HERE]]"
      CVAT_POSTGRES_USER: "[[ GOOGLE CLOUD USER HERE]]"      
      CVAT_POSTGRES_PASSWORD: "[[ GOOGLE CLOUD USER PASSWORD HERE]]"       
```
The file has been changed in the following ways:

 - removing the container postgres database (cvat_db)
 - adding the mounts to data to be in the (backed up) disk partition
 - updating the database environment credentials to be what you see above

Note that the host should be the full cloud sql connection name, and not the ip
address of the instance. Next, let's create the database (and of course obtain those credentials!) It's
fairly straight forward if you go to [the google cloud sql interface](https://console.cloud.google.com/sql/instances?authuser=1). 
You'll want to choose postgresql, an obvious instance name that says "this is for cvat and postgres" (e.g., cvat-development-postgres)
and then also select the database to be in the same zone as the instance. The defaults are fairly good -
the instance starts at a small size (10GB) but scales automatically. Automatic backups are also
enabled. We don't make the instance highly available, zone wise, because it's not going to be used
beyond a single lab. Note that the docker-compose file was using postgres 10, so that's also what we will use.

Finally, you'll want to ensure that your compute engine service account has the SQL Admin role added,
and you can find instructions for that [here](https://cloud.google.com/sql/docs/postgres/connect-compute-engine).
Since we have a reserved ip address, we can add the instance to be allowed to connect - [follow the instructions here](https://cloud.google.com/sql/docs/postgres/configure-ip#add).

**Note**: I've had this happen to me twice, but the password generated at the creation of the database
then doesn't work (you'll get a permissions / password denied) error. You can navigate to the
"Users" sidebar of Cloud managed sql to update it to what it should have already been.

Finally, on your instance (where you can connect) you'll want to create your databsae name.

```bash
sudo apt-get install postgresql-client
psql -h [[DATABASE_IP_ADDRESS]] -U postgres
CREATE DATABASE [[DATABASE_NAME]];
\q
```

If you don't do this, you'll get an error that the database name you specified does not exist.

#### Disabling Registration

Since this is open, we don't want to let anyone on the internet register.
We do this by way of binding the cvat code directory, and commenting out
this line:

```
        #path('register', RegisterView.as_view(), name='rest_register'),

```
Although the registration form can appear as a view, it will issue an error
to the server. The cvat team is currently [working on this](https://github.com/openvinotoolkit/cvat/issues/1283).

### Building Containers

Once the docker-compose file is updated and the database is ready, we can build containers!

```bash 
docker-compose build
```

And then bring them up!

```bash
docker-compose up -d
```

You can then see the containers!

```bash
docker-compose ps
```
And verify the mounts are working, you should see folders created on the host:

```bash
$ ls /mnt/disks/data/
data  keys  logs  lost+found  models
```

You should then create a superuser, likely for yourself and anyone else that needs that
kind of admin access.

```
$ docker exec -it cvat bash 
python3 ~/manage.py createsuperuser
# enter interactively
```

### Preparing for HTTPS

Since we will want to deploy a server with https, it makes sense to now look into how
you might want to obtain a domain. After you register the domain, we will use let's encrypt to
register it. Complete instructions are provided in [certificates.md](certificates.md).
Remember to create a docker-compose-override.yml file to allow for external access to your
application, and likely you want to bring down and remove the older containers first.


```bash
docker-compose stop
docker-compose rm
```

And then when you start, use it!

```bash
docker-compose -f docker-compose.yml -f docker-compose-override.yml up -d
```

You'll want to reference both these files when you issue commands, for the future.
It might be easier at some point to combine them into one!
