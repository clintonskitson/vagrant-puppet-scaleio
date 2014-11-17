**Summary - vagrant-puppet-scaleio**

The primary purpose of the Puppet module is to create automation necessary for an Operations team to install and configure ScaleIO nodes using Puppet. As an additional benefit, we provided as a valuable example of Vagrantfiles which allow rapid deployment pre-defined Puppet environments to configure ScaleIO deployments on a laptop or through other Vagrant providers. The project allows for a flexible ScaleIO environment serving for production, demos, or development.

With a working Puppet environment of server and agents deployed, it is possible to deploy pre-defined massive ScaleIO clusters across any cloud environment in minutes with this module.

**Puppet Module Repository**

For reference you can review the Puppet module repos at <a href="https://github.com/emccode/puppet-scaleio">Github</a> and <a href="https://forge.puppetlabs.com/emccode/scaleio/readme">Puppet Forge</a>. 

**Features**

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

