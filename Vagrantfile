VM_BOX      = "bento/ubuntu-22.04"
VM_PROVIDER = "virtualbox"

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false

# --------------- RANCHER ---------------
  RANCHER_VM_NAME   = "rancher"
  RANCHER_VM_CPUs   = 4
  RANCHER_VM_MEMORY = 4096 

  config.vm.define RANCHER_VM_NAME do |rancher|
      rancher.vm.hostname = RANCHER_VM_NAME
      rancher.vm.box     = VM_BOX
      rancher.vm.network "private_network", ip: "10.10.172.10"
      rancher.vm.network "public_network", bridge: "en0"
   
    rancher.vm.provider VM_PROVIDER do |vb|
      vb.name   = RANCHER_VM_NAME
      vb.cpus   = RANCHER_VM_CPUs
      vb.memory = RANCHER_VM_MEMORY 
    end
   
    rancher.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install htop -y
      curl -fsSL https://get.docker.com | sh
      systemctl enable --now docker
      usermod -aG docker vagrant 
      docker run -d \
        --restart=unless-stopped \
        -p 80:80 -p 443:443 \
        -v /opt/rancher:/var/lib/rancher \
        --name rancher \
        --privileged \
        rancher/rancher:stable 
    SHELL
  end
# --------------- NFS Server --------------
  NFS_SERVER_NAME   = "nfsserver"
  NFS_SERVER_CPUs   = 1
  NFS_SERVER_MEMORY = 512

  config.vm.define NFS_SERVER_NAME do |nfsserver|
      nfsserver.vm.hostname = NFS_SERVER_NAME
      nfsserver.vm.box = VM_BOX
      nfsserver.vm.network "private_network", ip: "10.10.172.20"

    nfsserver.vm.provider VM_PROVIDER do |vb|
      vb.name   = NFS_SERVER_NAME
      vb.cpus   = NFS_SERVER_CPUs
      vb.memory = NFS_SERVER_MEMORY 
    end

    nfsserver.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install htop nfs-kernel-server -y
      mkdir -p /nfs
      chown nobody:nogroup /nfs
      echo "/nfs *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee /etc/exports
      sudo systemctl restart nfs-kernel-server
    SHELL
  end

# --------------- NODES ---------------
  NODE_CPUs   = 2
  NODE_MEMORY = 1024

  (1..3).each do |i|
    node_name = "node#{i}"

    config.vm.define node_name do |node|
        node.vm.hostname = node_name
        node.vm.box      = VM_BOX
        node.vm.network "private_network", ip: "10.10.172.3#{i}"
      
      node.vm.provider VM_PROVIDER do |vb|
        vb.name   = node_name
        vb.cpus   = NODE_CPUs
        vb.memory = NODE_MEMORY
      end
      
      node.vm.provision "shell", inline: <<-SHELL
        apt-get update
        apt-get install htop -y
        echo "10.10.172.10 rancher.int" >> /etc/hosts
        curl -fsSL https://get.docker.com | sh
        systemctl enable --now docker
        usermod -aG docker vagrant
      SHELL
    end
  end
# --------------------------------------
end
