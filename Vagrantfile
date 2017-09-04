VAGRANTFILE_API_VERSION = "2"

# VirtualBox settings
vmmemory = 512
numcpu = 1

num_of_nginx = 1
num_of_managers = 1
num_of_workers = 1

nginx_instances = []
manager_instances = []
worker_instances = []


(1..num_of_nginx).each do |n| 
  nginx_instances.push({:id => n, :name => "nginx-#{n}", :ip => "192.168.10.#{n+1}"})
end

(1..num_of_managers).each do |n| 
  manager_instances.push({:id => n, :name => "manager-#{n}", :ip => "192.168.10.#{n+1+num_of_nginx}"})
end

(1..num_of_workers).each do |n| 
  worker_instances.push({:id => n, :name => "worker-#{n}", :ip => "192.168.10.#{n+1+num_of_nginx+num_of_managers}"})
end


File.open("./hosts", 'w') { |file| 
  nginx_instances.each do |i|
    file.write("#{i[:ip]} #{i[:name]} #{i[:name]}\n")
  end
  manager_instances.each do |i|
    file.write("#{i[:ip]} #{i[:name]} #{i[:name]}\n")
  end
  worker_instances.each do |i|
    file.write("#{i[:ip]} #{i[:name]} #{i[:name]}\n")
  end
}


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.provider "virtualbox" do |v|
    v.memory = vmmemory
  	v.cpus = numcpu
  end

  config.vm.synced_folder ".", "/vagrant"

  config.vm.provision :docker

  nginx_instances.each do |nginx|
    config.vm.define nginx[:name] do |i|
      i.vm.hostname = nginx[:name]
      i.vm.network "private_network", ip: "#{nginx[:ip]}"
      i.vm.provision "shell", path: "./provision.sh"

      if File.file?("./hosts") 
        i.vm.provision "file", source: "hosts", destination: "/tmp/hosts"
        i.vm.provision "shell", inline: "cat /tmp/hosts >> /etc/hosts", privileged: true
      end 

      File.open("./playbooks/host_vars/nginx-#{nginx[:id]}", 'w') { |file| 
        file.write("ip: #{nginx[:ip]}\n")
        if(nginx[:id] == 1)
          file.write("swarm_leader: true")
        else 
          file.write("swarm_leader_ip: #{nginx_instances[0][:ip]}")
        end
      }
    end
  end
    
  manager_instances.each do |manager|
    config.vm.define manager[:name] do |i|
      i.vm.hostname = manager[:name]
      i.vm.network "private_network", ip: "#{manager[:ip]}"
      i.vm.provision "shell", path: "./provision.sh"
 
      if File.file?("./hosts") 
        i.vm.provision "file", source: "hosts", destination: "/tmp/hosts"
        i.vm.provision "shell", inline: "cat /tmp/hosts >> /etc/hosts", privileged: true
      end 

      File.open("./playbooks/host_vars/manager-#{manager[:id]}", 'w') { |file| 
        file.write("ip: #{manager[:ip]}\n")
        if(manager[:id] == 1)
          file.write("swarm_leader: true")
        end
      }
    end
  end 

  worker_instances.each do |worker| 
    config.vm.define worker[:name] do |i|
      i.vm.hostname = worker[:name]
      i.vm.network "private_network", ip: "#{worker[:ip]}"
      i.vm.provision "shell", path: "./provision.sh"

      if File.file?("./hosts") 
        i.vm.provision "file", source: "hosts", destination: "/tmp/hosts"
        i.vm.provision "shell", inline: "cat /tmp/hosts >> /etc/hosts", privileged: true
      end 

      File.open("./playbooks/host_vars/worker-#{worker[:id]}", 'w') { |file| 
        file.write("ip: #{worker[:ip]}\n")
        file.write("swarm_leader_ip: #{manager_instances[0][:ip]}")
      }

      # Run after all boxes are setup
      if worker[:name] == worker_instances[num_of_workers-1][:name]
        i.vm.provision "ansible" do |ansible|
          ansible.playbook = "playbooks/main.yml"
          ansible.limit = "all"
          ansible.extra_vars = {
            swarm_iface: "eth1"
          }
          ansible.groups = {
            "nginx"   => ["nginx-1"],
            "manager" => ["manager-1"],
            "worker"  => ["worker-1"],
          }
        end
      end
    end 
  end
end