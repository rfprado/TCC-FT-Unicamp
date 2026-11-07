Vagrant.configure("2") do |config|
  # Provisionamento da box base do Rocky Linux 9, conforme catálogo oficial de boxes públicas disponível em https://portal.cloud.hashicorp.com/vagrant/discover
  box_image = "bento/rockylinux-9"
  #box_version = "1.0"
  
  # Procedimento alternativo para download da Box Oficial via site oficial do Rocky Linux 10 em sua última versão (manter comentado caso use a box do bento)
  # config.vm.box = "rockylinux/10"
  # config.vm.box_url = "https://dl.rockylinux.org/pub/rocky/10/images/x86_64/Rocky-10-Vagrant-Vbox.latest.x86_64.box"

  # Configuração do Nó 1: ceph-node01 (Bootstrap, MON, MGR, OSDs, Admin)
  config.vm.define "ceph-node01" do |node|
    node.vm.box = box_image
	#node.vm.box_version = box_version
    node.vm.hostname = "ceph-node01"    
    node.vm.network "private_network", ip: "192.168.56.10"
    node.vm.network "private_network", ip: "192.168.70.10", virtualbox__intnet: "ceph-cluster-net"
    node.vm.provider "virtualbox" do |vb|
      vb.name = "ceph-node01"
      vb.memory = 4096
      vb.cpus = 2
      # Criação de discos adicionais para OSDs (10GB cada)
      vb.customize ['createhd', '--filename', 'ceph-node01_disk2.vdi', '--size', '10240'] unless File.exist?('ceph-node01_disk2.vdi')
      vb.customize ['createhd', '--filename', 'ceph-node01_disk3.vdi', '--size', '10240'] unless File.exist?('ceph-node01_disk3.vdi')
      vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', '1', '--device', '0', '--type', 'hdd', '--medium', 'ceph-node01_disk2.vdi']
      vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', '2', '--device', '0', '--type', 'hdd', '--medium', 'ceph-node01_disk3.vdi']
    end
  end

  # Configuração do Nó 2: ceph-node02 (MON, MGR, OSDs, RGW)
  config.vm.define "ceph-node02" do |node|
    node.vm.box = box_image
	#node.vm.box_version = box_version
    node.vm.hostname = "ceph-node02"    
    node.vm.network "private_network", ip: "192.168.56.11"
    node.vm.network "private_network", ip: "192.168.70.11", virtualbox__intnet: "ceph-cluster-net"
    node.vm.provider "virtualbox" do |vb|
      vb.name = "ceph-node02"
      vb.memory = 3072
      vb.cpus = 2
      vb.customize ['createhd', '--filename', 'ceph-node02_disk2.vdi', '--size', '10240'] unless File.exist?('ceph-node02_disk2.vdi')
      vb.customize ['createhd', '--filename', 'ceph-node02_disk3.vdi', '--size', '10240'] unless File.exist?('ceph-node02_disk3.vdi')
      vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', '1', '--device', '0', '--type', 'hdd', '--medium', 'ceph-node02_disk2.vdi']
      vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', '2', '--device', '0', '--type', 'hdd', '--medium', 'ceph-node02_disk3.vdi']
    end
  end

  # Configuração do Nó 3: ceph-node03 (MON, OSDs, MDS, RGW)
  config.vm.define "ceph-node03" do |node|
    node.vm.box = box_image
	#node.vm.box_version = box_version
    node.vm.hostname = "ceph-node03"    
    node.vm.network "private_network", ip: "192.168.56.12"
    node.vm.network "private_network", ip: "192.168.70.12", virtualbox__intnet: "ceph-cluster-net"
    node.vm.provider "virtualbox" do |vb|
      vb.name = "ceph-node03"
      vb.memory = 3072
      vb.cpus = 2
      vb.customize ['createhd', '--filename', 'ceph-node03_disk2.vdi', '--size', '10240'] unless File.exist?('ceph-node03_disk2.vdi')
      vb.customize ['createhd', '--filename', 'ceph-node03_disk3.vdi', '--size', '10240'] unless File.exist?('ceph-node03_disk3.vdi')
      vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', '1', '--device', '0', '--type', 'hdd', '--medium', 'ceph-node03_disk2.vdi']
      vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', '2', '--device', '0', '--type', 'hdd', '--medium', 'ceph-node03_disk3.vdi']
    end
  end

  # Configuração do Nó Cliente: ceph-client
  config.vm.define "ceph-client" do |node|
    node.vm.box = box_image
	#node.vm.box_version = box_version
    node.vm.hostname = "ceph-client"    
    node.vm.network "private_network", ip: "192.168.56.20"
    node.vm.provider "virtualbox" do |vb|
      vb.name = "ceph-client"
      vb.memory = 2048
      vb.cpus = 1
    end
  end

  # Provisionamento Geral, Comum e Básico para Todos os Nós (Hosts, NTP, Ferramentas)
  config.vm.provision "shell", inline: <<-SHELL
    # 1. Configurar /etc/hosts
    cat <<EOF >> /etc/hosts
192.168.56.10 ceph-node01
192.168.56.11 ceph-node02
192.168.56.12 ceph-node03
192.168.56.20 ceph-client
EOF

    # 2. Atualizar o sistema e instalar dependências compatíveis com RHEL/Rocky Linux (DNF)
    dnf update -y
    dnf install -y lvm2 podman chrony python3 curl net-tools 

    # 3. Configurar e habilitar Sincronização de Tempo (Chrony)
    systemctl start chronyd && systemctl enable --now chronyd && chronyc tracking
  SHELL
  
  # 2. Provisionamento Exclusivo do ceph-node01: Geração e Distribuição de Chaves SSH
  config.vm.provision "shell", inline: <<-SHELL
    if [ "$(hostname)" = "ceph-node01" ]; then
      echo "==> Configurando o acesso SSH sem senha no ceph-node01..."
      
      # Gera chave SSH caso não exista
      if [ ! -f /root/.ssh/id_rsa ]; then
        ssh-keygen -t rsa -b 4096 -N "" -f /root/.ssh/id_rsa
      fi

      # Necessário para distribuição da chave SSH
      dnf install -y sshpass
      
      # Ajuste de permissões do /etc/ssh/sshd, permitindo login como root
      sed -i 's/#PermissiveRootLogin yes/PermitRootLogin yes/' /etc/ssh/sshd_config
      sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
      systemctl restart sshd

      # Aqui iniciamos a distribuição da chave pública para todos os nós (incluindo ele próprio e o client)
      for host in ceph-node01 ceph-node02 ceph-node03 ceph-client; do
        echo "Copiando chave para $host..."
        ssh-keyscan -H $host >> /root/.ssh/known_hosts
        sshpass -p 'vagrant' ssh-copy-id -o StrictHostKeyChecking=no root@$host
      done
      
      echo "==> Chaves SSH configuradas com sucesso em todos os nós!"
    fi
  SHELL
  
end