# -*- mode: ruby -*-
# vi: set ft=ruby :

# This configuration requires Vagrant 1.5 or newer and two plugins:
#
#   vagrant plugin install vagrant-hosts        ~> 2.1.4
#   vagrant plugin install vagrant-auto_network ~> 1.0.0
#

Vagrant.require_version ">= 1.7.2"
require 'vagrant-hosts'
require 'vagrant-hostmanager'
require 'vagrant-auto_network'

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'virtualbox'

# these are the internal IPs, alternate IPs are auto-assigned using vagrant-auto_network
puppetmaster_nodes = {
  'puppetmaster' => {
    :ip => '192.168.50.9', :hostname => 'puppetmaster', :domain => 'scaleio.local', :memory => 512, :cpus => 1
  }
}

# these are the internal IPs, alternate IPs are auto-assigned using vagrant-auto_network
scaleio_nodes = {
  'tb' => { :ip => '192.168.50.11', :hostname => 'tb', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
  'mdm1' => { :ip => '192.168.50.12', :hostname => 'mdm1', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
  'mdm2' => { :ip => '192.168.50.13', :hostname => 'mdm2', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
}

# (optional) specify download path, or comment out with # and specify
# $version and $rpm_suffix in site.pp file
#download_scaleio = "ftp://ftp.emc.com/Downloads/ScaleIO/ScaleIO_RHEL6_Download.zip"
download_scaleio = ""

if download_scaleio != ""
  perform_download = <<-EOF
    wget -nv ftp://ftp.emc.com/Downloads/ScaleIO/ScaleIO_RHEL6_Download.zip -O /tmp/ScaleIO_RHEL6_Download.zip
    unzip -o /tmp/ScaleIO_RHEL6_Download.zip -d /tmp/scaleio/
    cp /tmp/scaleio/ScaleIO_*_RHEL6*/*.rpm /etc/puppet/modules/scaleio/files/.
    cp /tmp/scaleio/ScaleIO_*_Gateway_*_Download/*.rpm /etc/puppet/modules/scaleio/files/.
    version=`basename /tmp/scaleio/ScaleIO_*_RHEL6_Download/EMC-ScaleIO-mdm* .el6.x86_64.rpm | cut -d- -f4,5,6`
    sed -i "/\\$version = /c\\\\\\$version = \'$version\'" /etc/puppet/manifests/site.pp
  EOF
end


Vagrant.configure('2') do |config|
  config.ssh.insert_key = false
  config.hostmanager.enabled=true
  config.hostmanager.include_offline = true
  config.vm.define puppetmaster_nodes['puppetmaster'][:hostname] do |node|
    node_hash = puppetmaster_nodes['puppetmaster']
    node.vm.box = 'puppetlabs/centos-7.0-64-nocm'
    node.vm.hostname = "#{node_hash[:hostname]}.#{node_hash[:domain]}"
    node.vm.provider "virtualbox" do |vb|
      vb.memory = node_hash[:memory] || 1024
      vb.cpus = node_hash[:cpus] || 1
    end

    node.vm.network :private_network, :ip => node_hash[:ip]
    node.vm.network :private_network, :auto_network => true

    # Use vagrant-hosts to add entries to /etc/hosts for each virtual machine
    # in this file.
    node.vm.provision :hosts

    bootstrap_script = <<-EOF
    if which puppet > /dev/null 2>&1; then
      echo 'Puppet Installed.'
    else
      yum remove -y firewalld && yum install -y iptables-services && iptables --flush
      echo 'Installing Puppet Master.'
      rpm -ivh http://yum.puppetlabs.com/el/7/products/x86_64/puppetlabs-release-7-10.noarch.rpm
      yum --nogpgcheck -y install puppet-server
      echo '*.#{node_hash[:domain]}' > /etc/puppet/autosign.conf
      puppet module install puppetlabs-stdlib
      puppet module install puppetlabs-firewall
      puppet module install puppetlabs-java
      puppet module install emccode-scaleio
      cp -Rf /opt/puppet/* /etc/puppet/.
      #{perform_download}
      puppet config set --section main parser future
      puppet config set --section master certname puppetmaster.#{node_hash[:domain]}
      /usr/bin/puppet resource service iptables ensure=stopped enable=false
      /usr/bin/puppet resource service puppetmaster ensure=running enable=true
    fi
    EOF
    node.vm.provision :shell, :inline => bootstrap_script
    node.vm.synced_folder "puppet", "/opt/puppet"
  end


  scaleio_nodes.each do |node_name,value|
    config.vm.provision :hosts do |provisioner|
      provisioner.add_host puppetmaster_nodes['puppetmaster'][:ip],
      ["#{puppetmaster_nodes['puppetmaster'][:hostname]}.#{puppetmaster_nodes['puppetmaster'][:domain]}","#{puppetmaster_nodes['puppetmaster'][:hostname]}"]
    end

    config.vm.define node_name do |node|
      node_hash = scaleio_nodes[node_name]
      node.vm.box = 'puppetlabs/centos-7.0-64-nocm'
      node.vm.hostname = "#{node_hash[:hostname]}.#{node_hash[:domain]}"
      node.vm.provider "virtualbox" do |vb|
        vb.memory = node_hash[:memory] || 1024
        vb.cpus = node_hash[:cpus] || 1
      end

      node.vm.network :private_network, :ip => node_hash[:ip]
      node.vm.network :private_network, :auto_network => true

      #node.vm.provision :hosts

      # Set up Puppet Agent to automatically connect with Puppet master
      bootstrap_script = <<-EOF
      ln -s /bin/rpm /usr/bin/rpm &> /dev/null
      if which puppet > /dev/null 2>&1; then
        echo 'Puppet Installed.'
      else
        yum remove -y firewalld && yum install -y iptables-services && iptables --flush
        echo 'Installing Puppet Agent.'
        rpm -ivh http://yum.puppetlabs.com/el/7/products/x86_64/puppetlabs-release-7-10.noarch.rpm
        yum --nogpgcheck -y install puppet
        puppet config set --section main server #{puppetmaster_nodes['puppetmaster'][:hostname]}.#{puppetmaster_nodes['puppetmaster'][:domain]}
        puppet agent -t --detailed-exitcodes || [ $? -eq 2 ]
      fi

      EOF
      node.vm.provision :shell, :inline => bootstrap_script

    end
  end



end
