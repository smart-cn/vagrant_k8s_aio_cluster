# -*- mode: ruby -*-
# vi: set ft=ruby :

CLUSTER_PREFIX = "k8s-AIO"
FILE_TO_NFS_DISK = "nfs_disk.vmdk"
CPUS_NUMBER = 4
CPU_LIMIT = 90
MEMORY_LIMIT=10240
OS_IMAGE = "ubuntu"
BRIDGE_ENABLE = false
BRIDGE_ETH = "eno1"
PRIVATE_SUBNET = "172.18.8"
IP_SHIFT = 10
UBUNTU_IMAGE = "ubuntu/jammy64"
CENTOS_IMAGE = "generic/centos9s"
NFS_DISKSIZE="500GB"

def set_vbox(vb, config, name)
  vb.gui = false
  vb.customize ["modifyvm", :id, "--cpuexecutioncap", "#{CPU_LIMIT}"]
  vb.customize ['modifyvm', :id, "--graphicscontroller", "vmsvga"]
  vb.customize ["modifyvm", :id, "--vram", "4"]
  vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  vb.memory = MEMORY_LIMIT + 250
  vb.cpus = CPUS_NUMBER
end

def set_libvirt(lv, config, name)
  lv.nested = true
  lv.volume_cache = "none"
  lv.uri = "qemu+unix:///system"
  lv.memory = MEMORY_LIMIT
  lv.cpus = CPUS_NUMBER
end

def set_hyperv(hv, config, name)
  hv.memory = MEMORY_LIMIT
  hv.cpus = CPUS_NUMBER
end

Vagrant.configure("2") do |config|
  config.vm.provider "hyperv"
  config.vm.provider "virtualbox"
  config.vm.provider "libvirt"
  config.vbguest.auto_update = false if Vagrant.has_plugin?("vagrant-vbguest")
  config.vm.define "#{CLUSTER_PREFIX}-cluster-AIO" do |n|
    n.disksize.size = "#{NFS_DISKSIZE}"
    if (OS_IMAGE == "ubuntu") then n.vm.box = UBUNTU_IMAGE; end
    if (OS_IMAGE == "centos") then n.vm.box = CENTOS_IMAGE; end
    name = "#{CLUSTER_PREFIX}-cluster-AIO"
    n.vm.hostname = name
    ip_addr = "#{PRIVATE_SUBNET}.2"
    n.vm.network :private_network, ip: "#{ip_addr}",  auto_config: true
    if BRIDGE_ENABLE && BRIDGE_ETH.to_s != ''
      n.vm.network "public_network", bridge: BRIDGE_ETH
    end
    n.vm.synced_folder "scripts/", "/home/vagrant/scripts"
    
    # Configure virtualbox provider
    n.vm.provider :virtualbox do |vb, override|
      vb.name = "#{n.vm.hostname}"
      set_vbox(vb, override, name)
    end

    # Configure libvirt provider
    n.vm.provider :libvirt do |lv, override|
      lv.host = "#{n.vm.hostname}"
      set_libvirt(lv, override, name)
    end

    # Configure hyperv provider
    n.vm.provider :hyperv do |hv, override|
      hv.vmname = "#{n.vm.hostname}"
      set_hyperv(hv, override, name)
    end
    
    n.vm.provision "shell", inline: <<-SHELL
      echo "Disabling firewalld"
      (systemctl stop firewalld && systemctl disable firewalld) || echo "Firewalld was not found, disabling skipped"
      echo "Allowing SSH login with password"
      sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config    
      systemctl restart sshd.service
      echo "Disabling IPV6"
      echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
      echo "net.ipv6.conf.all.disable_ipv6=1" >> /etc/sysctl.conf
      echo "net.ipv6.conf.default.disable_ipv6=1" >> /etc/sysctl.conf
      echo "net.ipv6.conf.lo.disable_ipv6=1" >> /etc/sysctl.conf
      sysctl -p
      echo "Detecting best mirror"
      apt update
      apt install python3-pip -y
      pip3 install apt-smart
      apt-smart --auto-change-mirror
      echo "Updating packages"
      (apt update -y; apt upgrade -y) || (yum update -y) || (echo "Unsupported OS, can not update packages, exiting" && exit 255)
      echo "Installing required NFS utils"
      (apt install -y nfs-common mc) || (yum install -y nfs-utils mc) || (echo "Unsupported OS, can not install required NFS packages, exiting" && exit 255)
    SHELL

    n.vm.network "forwarded_port", guest: 6443, host: 26443, protocol: "tcp"
    n.vm.network "forwarded_port", guest: 6443, host: 26443, protocol: "udp"
    n.vm.provision "shell", preserve_order: true, inline: <<-SHELL
      echo "Starting K8s cluster setup"
      (apt install -y sshpass) || (yum install -y sshpass) || (pkg install -y sshpass) || (echo "Unsupported OS, can not install required packages, exiting" && exit 255)
      echo "K8s cluster setup finished!!"
#      sshpass -p "vagrant" ssh -o StrictHostKeyChecking=no vagrant@"#{PRIVATE_SUBNET}.#{IP_SHIFT}" "./scripts/all-in-one-provisioner.sh"
    SHELL
    n.vm.network "forwarded_port", guest: 80, host: 80, protocol: "tcp"
    n.vm.network "forwarded_port", guest: 80, host: 80, protocol: "udp"
    n.vm.network "forwarded_port", guest: 443, host: 443, protocol: "tcp"
    n.vm.network "forwarded_port", guest: 443, host: 443, protocol: "udp"
  end

end
