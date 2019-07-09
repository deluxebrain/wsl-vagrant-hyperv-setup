# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANT_API_VERSION = "2"

if !ENV['APP_NAME']
    abort("FATAL: Environment not loaded")
    exit
end

# System install
$system_install_script = <<-SCRIPT
export DEBIAN_FRONTEND=noninteractive
apt-get update -qq
SCRIPT

Vagrant.configure(VAGRANT_API_VERSION) do |config|
    config.vm.box = "bento/ubuntu-18.04"    
    config.vm.hostname = ENV['APP_NAME']    
          
    # Fix `stdin: is not a tty` warning
    # https://github.com/hashicorp/vagrant/issues/1673
    config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"

    # Sytem install (root)
    config.vm.provision "shell",
        inline: $system_install_script,
        privileged: true 
   
    # Hyper-V
    config.vm.provider "hyperv" do |hv, override|
        hv.vmname = config.vm.hostname
        hv.memory = 1024
        hv.cpus = 1
        hv.vm_integration_services = {
            guest_service_interface: true,
            time_synchronization: true
        }    
    end  
end
