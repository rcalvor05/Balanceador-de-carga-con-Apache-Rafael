Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"

  config.vm.define "balanceador" do |b|
    b.vm.hostname = "RafaelCalvoRomeroBalanceador"
    b.vm.network "public_network", bridge: "enp0s31f6", ip: "192.168.18.200"
    b.vm.network "private_network", ip: "10.10.1.10", virtualbox__intnet: "red_web"
    b.vm.provider "virtualbox" do |vb|
      vb.name = "RafaelCalvoRomeroBalanceador"
      vb.memory = 512
    end
  end

  (1..2).each do |i|
    config.vm.define "web#{i}" do |w|
      w.vm.hostname = "RafaelCalvoRomeroWeb#{i}"
      w.vm.network "private_network", ip: "10.10.1.1#{i}", virtualbox__intnet: "red_web"
      w.vm.network "private_network", ip: "10.10.2.1#{i}", virtualbox__intnet: "red_bbdd"
      w.vm.provider "virtualbox" do |vb|
        vb.name = "RafaelCalvoRomeroWeb#{i}"
        vb.memory = 512
      end
    end
  end

  config.vm.define "bbdd" do |d|
    d.vm.hostname = "RafaelCalvoRomeroBBDD"
    d.vm.network "private_network", ip: "10.10.2.20", virtualbox__intnet: "red_bbdd"
    d.vm.provider "virtualbox" do |vb|
      vb.name = "RafaelCalvoRomeroBBDD"
      vb.memory = 512
    end
  end
end
