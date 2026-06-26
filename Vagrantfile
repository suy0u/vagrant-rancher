VM_BOX      = "bento/ubuntu-22.04"
VM_PROVIDER = "virtualbox"
BRIDGE_IFACE = "wlp5s0"
RANCHER_VERSION = "v2.9.3"
RKE2_TOKEN = "MySecureRKE2Token2024"

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false

# --------------- RANCHER ---------------
  RANCHER_VM_NAME   = "rancher"
  RANCHER_VM_CPUs   = 4
  RANCHER_VM_MEMORY = 4096

  config.vm.define RANCHER_VM_NAME do |rancher|
    rancher.vm.hostname = RANCHER_VM_NAME
    rancher.vm.box      = VM_BOX
    rancher.vm.network "private_network", ip: "10.10.172.10"
    rancher.vm.network "public_network", bridge: BRIDGE_IFACE

    rancher.vm.provider VM_PROVIDER do |vb|
      vb.name   = RANCHER_VM_NAME
      vb.cpus   = RANCHER_VM_CPUs
      vb.memory = RANCHER_VM_MEMORY
    end

    rancher.vm.provision "shell", inline: <<-SHELL
      apt-get update -q
      apt-get install -y htop curl
      curl -fsSL https://get.docker.com | sh
      systemctl enable --now docker
      usermod -aG docker vagrant
      docker run -d \
        --restart=unless-stopped \
        -p 80:80 -p 443:443 \
        -v /opt/rancher:/var/lib/rancher \
        --name rancher \
        --privileged \
        rancher/rancher:#{RANCHER_VERSION}
      echo ""
      echo "================================================================"
      echo " Rancher UI: https://10.10.172.10"
      echo " Initial password:"
      echo "   vagrant ssh rancher -c 'docker logs rancher 2>&1 | grep Bootstrap'"
      echo "================================================================"
    SHELL
  end

# --------------- NFS Server ---------------
  NFS_SERVER_NAME   = "nfsserver"
  NFS_SERVER_CPUs   = 1
  NFS_SERVER_MEMORY = 1024

  config.vm.define NFS_SERVER_NAME do |nfsserver|
    nfsserver.vm.hostname = NFS_SERVER_NAME
    nfsserver.vm.box      = VM_BOX
    nfsserver.vm.network "private_network", ip: "10.10.172.20"

    nfsserver.vm.provider VM_PROVIDER do |vb|
      vb.name   = NFS_SERVER_NAME
      vb.cpus   = NFS_SERVER_CPUs
      vb.memory = NFS_SERVER_MEMORY
    end

    nfsserver.vm.provision "shell", inline: <<-SHELL
      apt-get update -q
      apt-get install -y htop nfs-kernel-server
      mkdir -p /nfs
      chown nobody:nogroup /nfs
      chmod 777 /nfs
      echo "/nfs 10.10.172.0/24(rw,sync,no_subtree_check,no_root_squash)" > /etc/exports
      exportfs -ra
      systemctl enable --now nfs-kernel-server

      echo "================================================================"
      echo " NFS Server started on 10.10.172.20"
      echo "================================================================"
    SHELL
  end

# --------------- NODES ---------------
  NODE_CPUs   = 2
  NODE_MEMORY = 4096

  (1..3).each do |i|
    node_name = "node#{i}"
    node_ip   = "10.10.172.3#{i}"

    config.vm.define node_name do |node|
      node.vm.hostname = node_name
      node.vm.box      = VM_BOX
      node.vm.network "private_network", ip: node_ip

      node.vm.provider VM_PROVIDER do |vb|
        vb.name   = node_name
        vb.cpus   = NODE_CPUs
        vb.memory = NODE_MEMORY
      end

      node.vm.provision "shell", name: "common", inline: <<-SHELL
        apt-get update -q
        apt-get install -y htop nfs-common curl netcat-openbsd
        echo "10.10.172.10 rancher.local" >> /etc/hosts
        echo "10.10.172.20 nfsserver"    >> /etc/hosts
        curl -fsSL https://get.docker.com | sh
        systemctl enable --now docker
        usermod -aG docker vagrant
      SHELL

      if i == 1
        # node1 — RKE2 Server (control-plane + etcd + worker)
        node.vm.provision "shell", name: "rke2-server", inline: <<-SHELL
          curl -sfL https://get.rke2.io | sh -
          mkdir -p /etc/rancher/rke2

          echo "token: #{RKE2_TOKEN}" > /etc/rancher/rke2/config.yaml
          echo "node-ip: #{node_ip}" >> /etc/rancher/rke2/config.yaml
          echo "tls-san:" >> /etc/rancher/rke2/config.yaml
          echo "  - #{node_ip}" >> /etc/rancher/rke2/config.yaml
          echo "  - 127.0.0.1"  >> /etc/rancher/rke2/config.yaml
          echo "disable-cloud-controller: true" >> /etc/rancher/rke2/config.yaml

          systemctl enable rke2-server.service
          systemctl start rke2-server.service

          mkdir -p /home/vagrant/.kube
          cp /etc/rancher/rke2/rke2.yaml /home/vagrant/.kube/config
          sed -i 's/127.0.0.1/#{node_ip}/' /home/vagrant/.kube/config
          chown -R vagrant:vagrant /home/vagrant/.kube
          echo 'export KUBECONFIG=/home/vagrant/.kube/config'  >> /home/vagrant/.bashrc
          echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin'   >> /home/vagrant/.bashrc

          echo ""
          echo "================================================================"
          echo " node1: RKE2 started on #{node_ip}:9345"
          echo "================================================================"
        SHELL
      else
        # node2, node3 — RKE2 agents
        node.vm.provision "shell", name: "rke2-agent", inline: <<-SHELL
          # Waiting for the RKE2 server (node1) to be ready
          echo "Waiting for the RKE2 server at 10.10.172.31:9345..."
          for attempt in $(seq 1 30); do
            nc -z 10.10.172.31 9345 && break
            echo "  attempt $attempt/30..."
            sleep 10
          done

          curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
          mkdir -p /etc/rancher/rke2

          echo "server: https://10.10.172.31:9345" > /etc/rancher/rke2/config.yaml
          echo "token: #{RKE2_TOKEN}" >> /etc/rancher/rke2/config.yaml
          echo "node-ip: #{node_ip}" >> /etc/rancher/rke2/config.yaml
          
          systemctl enable rke2-agent.service
          systemctl start rke2-agent.service

          echo ""
          echo "================================================================"
          echo " #{node_name}: RKE2 agent connected to 10.10.172.31"
          echo "================================================================"
        SHELL
      end
    end
  end
# --------------------------------------
end
