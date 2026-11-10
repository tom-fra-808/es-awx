Vagrant.configure("2") do |config|
  config.vm.box = "generic/debian12"
  config.vm.hostname = "awx-target"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 512
    vb.cpus = 1
  end
end
