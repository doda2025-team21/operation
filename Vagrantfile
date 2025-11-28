# Load .env file if exists
if File.exist?(".env")
  File.foreach(".env") do |line|
    key, value = line.strip.split("=", 2)
    ENV[key] = value if key && value
  end
end

# You can also manually set them here!
NUM_WORKERS   = ENV.fetch("NUM_WORKERS", "2").to_i   # number of worker nodes
CTRL_CPUS     = ENV.fetch("CTRL_CPUS", "1").to_i
CTRL_MEMORY   = ENV.fetch("CTRL_MEMORY", "4096").to_i  # MB
NODE_CPUS     = ENV.fetch("NODE_CPUS", "2").to_i
NODE_MEMORY   = ENV.fetch("NODE_MEMORY", "6144").to_i  # MB


Vagrant.configure("2") do |config|
  # All VMs use Ubuntu 24.04
  config.vm.box = "bento/ubuntu-24.04"

  # Do not replace default insecure key; makes Ansible/Vagrant work out of the box
  config.ssh.insert_key = false

  #------------------------
  # Controller node: ctrl
  #------------------------
  config.vm.define "ctrl" do |ctrl|
    ctrl.vm.hostname = "ctrl"

    # NIC 1: default NAT (for internet access) - created automatically by Vagrant

    # NIC 2: host-only network, fixed IP 192.168.56.100 (Step 2)
    ctrl.vm.network "private_network", ip: "192.168.56.100"

    ctrl.vm.provider "virtualbox" do |vb|
      vb.name   = "k8s-ctrl"
      vb.cpus   = CTRL_CPUS
      vb.memory = CTRL_MEMORY
    end
  end

  #------------------------
  # Worker nodes: node-1 .. node-N
  #------------------------
  (1..NUM_WORKERS).each do |i|
    config.vm.define "node-#{i}" do |node|
      node.vm.hostname = "node-#{i}"

      # NIC 1: default NAT

      # NIC 2: host-only, 192.168.56.101+ (Step 2)
      ip_octet = 100 + i         # node-1 => 101, node-2 => 102, ...
      node.vm.network "private_network", ip: "192.168.56.#{ip_octet}"

      node.vm.provider "virtualbox" do |vb|
        vb.name   = "k8s-node-#{i}"
        vb.cpus   = NODE_CPUS
        vb.memory = NODE_MEMORY
      end
    end
  end

  #----------------------------------------------------------
  # Ansible provisioning (Step 3 + Step 4)
  #----------------------------------------------------------
  # 说明：
  # - 不指定 inventory_path，Vagrant 会自动生成 inventory，
  #   里面有主机名 ctrl, node-1, node-2, ...
  # - 我们通过 hosts pattern 区分 playbook 的作用范围
  # - extra_vars 把 num_workers 传进去，后面可以用

  # 1) General playbook: runs on all nodes (Step 3: provision + Step 4: SSH keys)
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/general.yml"
    ansible.inventory_path = nil          # let Vagrant auto-generate
    ansible.limit = "all"
    ansible.extra_vars = {
      num_workers: NUM_WORKERS
    }
  end

  # 2) Controller-specific playbook (later steps will live here)
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/ctrl.yml"
    ansible.inventory_path = nil
    ansible.limit = "ctrl"
    ansible.extra_vars = {
      num_workers: NUM_WORKERS
    }
  end

  # 3) Worker-specific playbook (later steps will live here)
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/node.yml"
    ansible.inventory_path = nil
    ansible.limit = "node-*"
    ansible.extra_vars = {
      num_workers: NUM_WORKERS
    }
  end
end