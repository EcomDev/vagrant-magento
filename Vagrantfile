require 'json'

magentoJson = JSON.parse(File.read('magento.json'))

Vagrant.configure("2") do |config|
  if Vagrant.has_plugin?("vagrant-berkshelf")
    config.berkshelf.enabled = true
  end

  if Vagrant.has_plugin?("vagrant-omnibus")
    config.omnibus.chef_version = :latest
  end

  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.include_offline = true
  end

  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box

    config.cache.synced_folder_opts = {
        type: :nfs,
        mount_options: ['rw', 'tcp', 'nolock', 'async']
    }

    config.cache.auto_detect = false
    config.cache.enable :apt
    config.cache.enable :apt_lists
    config.cache.enable :apt_cacher
    config.cache.enable :gem
    config.cache.enable :yum
    config.cache.enable :pacman
    config.cache.enable :npm
  end

  if magentoJson['magento']['application'].key?('uid') && magentoJson['magento']['application']['uid'] === ':auto'
    magentoJson['magento']['application']['uid'] = Process.euid
  end

  # Box that is used for chef
  # Debian 7.4
  config.vm.box = 'opscode-debian-7.4'
  config.vm.box_url = 'http://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_debian-7.4_chef-provisionerless.box'
  # Possible options are:
  ## Ubuntu 12.04
  # config.vm.box = "opscode-ubuntu-12.04"
  # config.vm.box_url = "https://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_ubuntu-12.04_chef-provisionerless.box"
  ## CentOS 6.5
  # config.vm.box = "opscode-centos-6.5"
  # config.vm.box_url = "https://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_centos-6.5_chef-provisionerless.box"

  config.vm.network :private_network, ip: magentoJson['vm']['ip']
  config.vm.hostname = magentoJson['magento']['application']['name']

  magentoJson['vm']['mount_dirs'].each do |local, guest|
    config.vm.synced_folder local, guest, nfs: true, mount_options: ['rw', 'tcp', 'nolock', 'async']
  end

  host = RbConfig::CONFIG['host_os']

  # Give VM 1/4 system memory & access to all cpu cores on the host
  if host =~ /darwin/
    cpus = `sysctl -n hw.ncpu`.to_i
    # sysctl returns Bytes and we need to convert to MB
    mem = `sysctl -n hw.memsize`.to_i / 1024 / 1024 / 4
  elsif host =~ /linux/
    cpus = `nproc`.to_i
    # meminfo shows KB and we need to convert to MB
    mem = `grep 'MemTotal' /proc/meminfo | sed -e 's/MemTotal://' -e 's/ kB//'`.to_i / 1024 / 4
  else # sorry Windows folks, I can't help you
    cpus = magentoJson['vm']['cpu']
    mem = magentoJson['vm']['memory']
  end

  config.vm.provider :virtualbox do |vb|
    vb.customize ['modifyvm', :id, '--memory', mem, '--cpus', cpus]
    vb.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
  end

  config.vm.provision :chef_solo do |chef|
    chef.json = {
        magento: magentoJson['magento'],
        php: magentoJson[:php]
    }
    chef.run_list = magentoJson['recipes'].map {|v| "recipe[#{v}]"}
  end

  domain_aliases = [magentoJson['magento']['application']['main_domain']]
  domain_aliases << magentoJson['magento']['application']['domain_map'].keys
  if magentoJson['magento']['application']['domains'].is_a?(Array)
    domain_aliases << magentoJson['magento']['application']['domains']
  end

  domain_aliases.flatten!.uniq!
  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.aliases = domain_aliases
  end
end
