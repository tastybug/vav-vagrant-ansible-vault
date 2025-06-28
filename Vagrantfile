# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Base box configuration
  config.vm.box = "rockylinux/9"
  config.vm.box_version = "4.0.0"

  # SSH configuration
  config.ssh.insert_key = false
  config.ssh.private_key_path = ["~/.vagrant.d/insecure_private_key", "~/.ssh/ansible"]
  config.vm.provision "file", source: "~/.ssh/ansible.pub", destination: "~/.ssh/authorized_keys"

  # Global VM settings
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 1
  end

  # Common provisioning script
  config.vm.provision "shell", inline: <<-SHELL
	sudo systemctl start firewalld
    # Update system
    #sudo dnf update -y
    
    # Install required packages
    sudo dnf install -y podman systemd dbus python3-pip git wget curl
    
    sudo loginctl enable-linger vagrant
    
    # Configure firewall for Vault ports
	sudo systemctl start firewalld
    sudo firewall-cmd --permanent --add-port=8200/tcp
    sudo firewall-cmd --permanent --add-port=8201/tcp
    sudo firewall-cmd --reload
    
    # Ensure SSH is properly configured
    sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
    sudo systemctl restart sshd
    
    # Set up podman for the vagrant user
    sudo -u vagrant podman system migrate || true
	podman run hello-world
  SHELL

  # Define the three Vault nodes
  (1..3).each do |i|
    config.vm.define "vault#{i}" do |node|
      node.vm.hostname = "vault#{i}"
      node.vm.network "private_network", ip: "192.168.60.#{20 + i}"
      
      # Port forwarding for Vault UI (only for vault1)
      if i == 1
        node.vm.network "forwarded_port", guest: 8200, host: 8200, host_ip: "127.0.0.1"
      end
      
      node.vm.provider "virtualbox" do |vb|
        vb.name = "vault#{i}"
        vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
      end

      # Add entries to /etc/hosts for all vault nodes
      node.vm.provision "shell", inline: <<-SHELL
        # Add all vault nodes to /etc/hosts
        echo "192.168.60.21 vault1" >> /etc/hosts
        echo "192.168.60.22 vault2" >> /etc/hosts
        echo "192.168.60.23 vault3" >> /etc/hosts
        
        # Ensure proper hostname resolution
        sudo hostnamectl set-hostname vault#{i}
      SHELL
    end
  end
end
