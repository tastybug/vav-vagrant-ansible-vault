# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Base box configuration
  config.vm.box = "rockylinux/9"
  config.vm.box_version = ">= 4.0.0"

  # SSH configuration
  config.ssh.insert_key = false
  config.ssh.private_key_path = ["~/.vagrant.d/insecure_private_key", "~/.ssh/id_rsa"]
  config.vm.provision "file", source: "~/.ssh/id_rsa.pub", destination: "~/.ssh/authorized_keys"

  # Global VM settings
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
    vb.linked_clone = true
  end

  # Common provisioning script
  config.vm.provision "shell", inline: <<-SHELL
    # Update system
    dnf update -y
    
    # Install required packages
    dnf install -y podman podman-compose python3-pip git wget curl
    
    # Enable podman socket for rootless containers
    systemctl --user enable podman.socket --now || true
    loginctl enable-linger vagrant
    
    # Configure firewall for Vault ports
    firewall-cmd --permanent --add-port=8200/tcp
    firewall-cmd --permanent --add-port=8201/tcp
    firewall-cmd --reload
    
    # Ensure SSH is properly configured
    sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
    systemctl restart sshd
    
    # Set up podman for the vagrant user
    sudo -u vagrant podman system migrate || true
    sudo -u vagrant systemctl --user enable podman.socket --now || true
  SHELL

  # Define the three Vault nodes
  (1..3).each do |i|
    config.vm.define "vault#{i}" do |node|
      node.vm.hostname = "vault#{i}"
      node.vm.network "private_network", ip: "192.168.56.#{10 + i}"
      
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
        echo "192.168.56.11 vault1" >> /etc/hosts
        echo "192.168.56.12 vault2" >> /etc/hosts
        echo "192.168.56.13 vault3" >> /etc/hosts
        
        # Ensure proper hostname resolution
        hostnamectl set-hostname vault#{i}
      SHELL
    end
  end
end