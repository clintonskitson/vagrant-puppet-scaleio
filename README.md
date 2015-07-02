**Summary - vagrant-puppet-scaleio**

This repository houses a Vagrantfile and Puppet configuration files that allow you to deploy and scale-up/out a ScaleIO storage environment.  It makes use of the ```puppet-scaleio``` module to perform all of the configuration and ```Vagrant``` to deploy a Puppet Master, configure Puppet with ScaleIO module, and then deploy additional defined ScaleIO nodes.

With this Vagrantfile, it is possible to deploy pre-defined ScaleIO clusters.  Further, the ScaleIO puppet module allows you to deploy massive descriptive ScaleIO clusters across any environment, physical or virtual in minutes.

**ScaleIO Downloads**
A new feature of this Vagrantfile is to download the latest ScaleIO RPM's automatically.  This is done automatically and can be disabled by editing the Vagrantfile and commenting out ```download_scaleio```.

**Simple Instructions**
- git clone https://github.com/emccode/vagrant-puppet-scaleio
- (optional) copy ScaleIO RPM's into puppet/modules/scaleio/files/
 - (optional) update puppet/manifests/site.pp file to make $version align to files
- vagrant up
- vagrant ssh mdm1
- ..scli away!

**Instructions on using REX-ray & Docker with ScaleIO**
[REX-ray](https://github.com/emccode/rexray) is a storage inspection package that automates the provisioning, mounting, and attachment of volumes from multiple storage services to ephemeral docker containers, which creates persistent data storage. This is solving one of the biggest problem we see today in the container space. Today, many people are mounting external volumes to hosts and using the `-v` switch to mount that volume to a container. Rexray automates this process by evaluating the storage systems. It can automatically detect the storage systems attached to the host, and can execute commands to the underlying storage system without user interaction. Rexray supports many storage platforms and we will demonstrate it today using ScaleIO. ScaleIO will be our direct attached storage system that replicates its data between all nodes in a cluster.

[REX-ray](https://github.com/emccode/rexray) can be installed as a simple go package, but to make this easy we will use this branch from `vagrant-puppet-scalio` repo from EMC {code} to have everything packaged in a simple vagrant environment. This particular branch has the experimental release of docker 1.7 and [REX-ray](https://github.com/emccode/rexray) installed for us.
 
Clone the branch that has the experimental version of docker along with rexray integration and `vagrant up`
```
- git clone -b docker_1.7rc3_experimental_rexray https://github.com/emccode/vagrant-puppet-scaleio.git
- cd vagrant-puppet-scaleio
- vagrant up
- vagrant ssh mdm1
- sudo su
- rexray get-instance
- rexray new-volume --volumename=testing1 --size=8 (test rexray with new-volume)
- docker run --volume-driver=rexray -ti -v newvol:/newvol busybox (create another volume directly from docker)
- touch /newvol/test
- exit (exit from container)
- exit (exit from mdm1)
- vagrant ssh mdm2
- sudo su
- docker run --volume-driver=rexray -ti -v newvol:/newvol busybox (attach volume to anther OS and container)
- ls /newvol (look for test file)
```

Vagrant will provision a puppet-master that holds the configurations for the tie breaker and mdm machines. these three hosts have a direct attached disk and will form a scaleio cluster where all data will be replicated between the three hosts. this process usually takes about 5-10 minutes. SSH into `mdm1` and get root access
```
vagrant ssh mdm1
sudo su
```

The first command to run with rexray is the `rexray get-instance` command. This command will do an introspection of the storage environment. In our case, it has determined the only available storage platform is Scale IO and sets that as the default. 
```
[root@mdm1 vagrant]# rexray get-instance
- providername: scaleio
  instanceid: b6c295d700000000
  region: ""
  name: ""
```

Create a customized volume `rexray new-volume --volumename=rexrayvol01 --size=24`
```
[root@mdm1 vagrant]# rexray new-volume --volumename=rexrayvol01 --size=24
name: rexrayvol01
volumeid: b5a167e400000001
availabilityzone: d263a0ed00000000
status: ""
volumetype: 91b4120f00000000
iops: 0
size: "24"
attachments: []
```

To verify it was created on ScaleIO, use `scli` to show it’s really there
```
[root@mdm1 vagrant]# scli --login --username admin --password Scaleio123
Logged in. User role is Administrator. System ID is 6647c808246dd8ea
[root@mdm1 vagrant]# scli --query_all_volumes
Query-all-volumes returned 2 volumes
Protection Domain d263a0ed00000000 Name: protection_domain1
Storage Pool 91b4120f00000000 Name: capacity
 Volume ID: b5a167e300000000 Name: volume1 Size: 8.0 GB (8192 MB) Mapped to 3 SDC Thick-provisioned
 Volume ID: b5a167e400000001 Name: rexrayvol01 Size: 24.0 GB (24576 MB) Unmapped Thick-provisioned
```

Create a new container specifying the `--volume-driver` flag and mounting our new volume: `docker run --name=h1c1 --volume-driver=rexray -ti -v rexrayvol01:/rexrayvol01 busybox`
```
[root@mdm1 vagrant]# docker run --name=h1c1 --volume-driver=rexray -ti -v rexrayvol01:/rexrayvol01 busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from busybox
cf2616975b4a: Pull complete
6ce2e90b0bc7: Pull complete
8c2e06607696: Already exists
busybox:latest: The image you are pulling has been verified. Important: image verification is a tech preview feature and should not be relied on to provide security.
Digest: sha256:38a203e1986cf79639cfb9b2e1d6e773de84002feea2d4eb006b52004ee8502d
Status: Downloaded newer image for busybox:latest
/ # 
```

Create a new file so we can see the data persists between containers event after they are stopped.
```
/ # ls
bin          etc          lib          linuxrc      mnt          proc         root         sbin         tmp          var
dev          home         lib64        media        opt          rexrayvol01  run          sys          usr
/ # touch /rexrayvol01/test
/ # ls /rexrayvol01
lost+found  test
/ # exit
```

SSH into `mdm2` and create a new docker container and attach the ScaleIO volume
```
[root@mdm2 vagrant]# docker run --name=h2c1 --volume-driver=rexray -ti -v rexrayvol01:/rexrayvol01 busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from busybox
cf2616975b4a: Pull complete
6ce2e90b0bc7: Pull complete
8c2e06607696: Already exists
busybox:latest: The image you are pulling has been verified. Important: image verification is a tech preview feature and should not be relied on to provide security.
Digest: sha256:38a203e1986cf79639cfb9b2e1d6e773de84002feea2d4eb006b52004ee8502d
Status: Downloaded newer image for busybox:latest
/ #
```

Do a `ls` on the mounted volume and there is the file that was originally created on `mdm1`.
```
/ # ls
bin          etc          lib          linuxrc      mnt          proc         root         sbin         tmp          var
dev          home         lib64        media        opt          rexrayvol01  run          sys          usr
/ # cd rexrayvol01/
/rexrayvol01 # ls
lost+found  test
/rexrayvol01 # exit
```

To create new volumes for us on the fly, specify an uncreated volume in the `docker run` command.
```
[root@mdm2 vagrant]# docker run --name=h2c2 --volume-driver=rexray -ti -v newvol:/newvol busybox
/ # touch /newvol/testfile
/ # exit

[root@mdm2 vagrant]# rexray get-volume
- name: volume1
  volumeid: b5a167e300000000
  availabilityzone: d263a0ed00000000
  status: ""
  volumetype: 91b4120f00000000
  iops: 0
  size: "8"
  attachments:
  - volumeid: b5a167e300000000
    instanceid: b6c295d800000001
    devicename: ""
    status: ""
  - volumeid: b5a167e300000000
    instanceid: b6c295d900000002
    devicename: /dev/scinia
    status: ""
  - volumeid: b5a167e300000000
    instanceid: b6c295d700000000
    devicename: ""
    status: ""
- name: rexrayvol01
  volumeid: b5a167e400000001
  availabilityzone: d263a0ed00000000
  status: ""
  volumetype: 91b4120f00000000
  iops: 0
  size: "24"
  attachments: []
- name: newvol
  volumeid: b5a167e500000002
  availabilityzone: d263a0ed00000000
  status: ""
  volumetype: 91b4120f00000000
  iops: 0
  size: "16"
  attachments: []
```

Delete Volumes using `rexray remove-volume`
```
[root@mdm1 vagrant]# rexray remove-volume --volumeid=b5a167e400000001
2015/06/18 14:38:56 Deleted Volume: b5a167e400000001
```

Or Delete Volumes using `docker rm -v <container id>`
```
[root@mdm1 vagrant]# docker rm -v f6d135bdb789
f6d135bdb789
```

**Puppet Module Repository**

For reference you can review the Puppet module repos at <a href="https://github.com/emccode/puppet-scaleio">Github</a> and <a href="https://forge.puppetlabs.com/emccode/scaleio/readme">Puppet Forge</a>.

**Puppet Module Features**

- Puppet Agent installs proper RPMs for components
- Configures firewall settings
- Sets required OS kernel SHM settings
- Sets /dev/shm temporary fs size
- Multiple component types per node (Tie-Breaker, MDM, SDS, SDC, GW, LIA, Call-Home)
- Configures MDM cluster services
- Configure multiple Protection Domains with Storage Pools
- Configure Storage Pools
- Multiple devices per SDS with per node definition of device paths, device size, storage pool
- Add Devices to SDSs
- Add Volumes
- Map Volumes to SDCs


**Documentation**  
<a href="http://puppet-scaleio-docs.readthedocs.org/en/latest/">Read The Docs</a>

**Contributors/Sources**
- Eoghan Kelleher
- Jonas Rosland
- Clint Kitson

**Contributing**  
We encourage the community to actively contribute to this module.  
- Fork the repository  
- Clone  
- Add original repository as upstream  
- Checkout new branch  
- Commit changes  
- Push to your repository  
- Issue pull request  

**Licensing**  
Licensed under the Apache License, Version 2.0 (the “License”); you may not use this file except in compliance with the License. You may obtain a copy of the License at <http://www.apache.org/licenses/LICENSE-2.0>  

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an “AS IS” BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.


**Support**  
Please file bugs and issues at the <a href="https://github.com/emccode/puppet-scaleio/issues">Github issues</a> page.  For more general discussions you can contact the EMC Code team at <a href="https://groups.google.com/forum/#!forum/emccode-users">Google Groups</a>.  The code and documentation are released with no warranties or SLAs and are intended to be supported through a community driven process.
