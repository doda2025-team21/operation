# Load .env file if exists
if File.exist?(".env")
  File.foreach(".env") do |line|
    key, value = line.strip.split("=", 2)
    ENV[key] = value if key && value
  end
end

# Create an empty inventory.cfg if it does not exist
unless File.exist?("inventory.cfg")
  File.write("inventory.cfg", "")
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
  # config.ssh.insert_key = false

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
  # Synced-folders
  #----------------------------------------------------------

  # the first parameter is a path to a directory on the host machine.
  # in the case that the path is relative, it is relative to the project root
  # the second parameter must be an absolute path of where to share the folder 
  # within the guest machine. 
  # this folder will be created (recursively, if it must) if it does not exist.
  # By default, vagrant mounts the synced folders with the owner/group set 
  # to the SSH user and any parent folders set to root
  # create: if true, the host path will be created if 
  # it does not exist. Defaults to false
  # owner: the user who should be the owner of this synced folder. 
  # by default this will be the SSH user. 
  # group: the group that will own the synced folder. 
  # by default this will bbe the SSH user. 
  # optionally you can use "root" but not needed
  # mount_options: a list of additional mount options to pass 
  # to the mount command

  config.vm.synced_folder "./shared", "/mnt/shared",
    create: true,
    owner: "vagrant", 
    group: "vagrant"
    # mount_options: []



  #----------------------------------------------------------
  # Generate inventory.cfg automatically
  #----------------------------------------------------------

  config.trigger.before :provision do |t|
    t.name = "generating inventory.cfg"
    t.ruby do |env_variable, triggering_machine|
      node_names = ["ctrl"] + (1..NUM_WORKERS).map {|i| "node-#{i}"}
      running = []
      node_names.each do |name|
        out = `vagrant status #{name}`
        running << name if out.include?("running")
      end

      ANSIBLE_INVENTORY_FILE = "inventory.cfg"

      File.open("#{ANSIBLE_INVENTORY_FILE}", 'w') do |f|
        f.write "[controllers]\n"
        if running.include?("ctrl")
          f.write "ctrl ansible_host=192.168.56.100 ansible_private_key_file=.vagrant/machines/ctrl/virtualbox/private_key\n"
        end 
        f.write "\n"
        
        f.write "[workers]\n"
        (1..NUM_WORKERS).each do |i|
          node = "node-#{i}"
          octet = 100 + i 
          ip =  "192.168.56.#{octet}"
          if running.include?(node)
            f.write "#{node} ansible_host=#{ip} ansible_private_key_file=.vagrant/machines/#{node}/virtualbox/private_key\n"
          end
        end 

        f.write "\n"

        f.write "[all:vars]\n"
        f.write "ansible_user=vagrant\n"
        f.write "ansible_python_interpreter=/usr/bin/python3\n"
        f.write "num_workers=#{NUM_WORKERS}\n"

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
    ansible.inventory_path = "inventory.cfg"          
    ansible.limit = "all"
    ansible.extra_vars = {
      num_workers: NUM_WORKERS
    }
  end

  # 2) Controller-specific playbook (later steps will live here)
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/ctrl.yml"
    ansible.inventory_path = "inventory.cfg"
    ansible.limit = "ctrl"
    ansible.extra_vars = {
      num_workers: NUM_WORKERS
    }
  end

  # 3) Worker-specific playbook (later steps will live here)
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/node.yml"
    ansible.inventory_path = "inventory.cfg"
    ansible.limit = "node-*"
    ansible.extra_vars = {
      num_workers: NUM_WORKERS
    }
  end
end