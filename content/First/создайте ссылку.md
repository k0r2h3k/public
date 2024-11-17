
Запуск 
```
powershell
cd G:\GOAD\ad\GOAD\providers\vmware\
vagrant.exe up
```

https://github.com/Orange-Cyberdefense/GOAD

To deploy on Windows we need a few steps over and above standard VMWare setup detailed in install_with_vmware.md.

## Prerequisites

- Tooling to install on Windows
  - [Vmware workstation](https://www.vmware.com/products/workstation-pro/workstation-pro-evaluation.html)
  - [Vagrant for Windows](https://developer.hashicorp.com/vagrant/install?product_intent=vagrant#Windows)
  - [Vmware utility driver](https://developer.hashicorp.com/vagrant/install/vmware)
  - Vagrant plugins:
    - vagrant-reload
    - vagrant-vmware-desktop
    - `vagrant plugin install vagrant-vmware-desktop --plugin-clean-sources --plugin-source https://rubygems.org`
    - `vagrant plugin install vagrant-reload --plugin-clean-sources --plugin-source https://rubygems.org`
- Kali or Ubuntu VM, installed inside VMWare Workstation
    - Ensure the VM has two NICs, one NAT/Bridged for Internet and a second in the same subnet as GOAD default setup which is `192.168.56.0` and `255.255.255.0` netmask via VMWare Workstation's Virtual Network Editor.
    - Install Ansible and Dependencies
    ```
    pip install --upgrade pip
    pip install ansible-core==2.12.6
    pip install pywinrm

    sudo apt install sshpass lftp rsync openssh-client
    git clone https://github.com/Orange-Cyberdefense/GOAD
    ```
    - Install Ansible requirements
        - drop into `GOAD/ansible` on Ubuntu/Kali VM and execute:
        `ansible-galaxy install -r ansible/requirements.yml`


## Setup VMs with Vagrant
Once pre-reqs have been installed, next thing to do is to deploy the baseline VMs with vagrant from cmd/PowerShell.

### Create the vms

- To create the VMs just run 

```powershell
powershell
cd D:\GOAD\ad\GOAD\providers\vmware
vagrant up
```

This will proceed to run through pulling down the five GOAD virtual machines. Once complete you can proceed to the next step which is deploying ansible to confirgure the VMs. 

### Deploy Ansible to Build VMs
Once VMs have all built with Vagrant, the next step is to hop into your Kali/Ubuntu VM and roll with running Ansible to configure them. To do this, navigate to the GOAD directory and run the goad.sh setup script as a standard user:

```
./goad.sh -t install -l GOAD -p vmware -m local -a
```

Provided you've done all the pre-req setup stages, this will run through the setup of all the VMs and configure them to the GOAD Ansible YML file specs.  

## Исправление ошибок

Проверка настроек сети
![[Pasted image 20241116210218.png]]

Загрузка образов ОС через VPN
https://app.vagrantup.com/StefanScherer/boxes/windows_2019
https://app.vagrantup.com/StefanScherer/boxes/windows_2016
https://portal.cloud.hashicorp.com/vagrant/discover/bento/ubuntu-18.04

Не обязательно если в папке в vagrant файлом есть скачанный box
```bash
cd D:\GOAD\ad\GOAD\providers\vmware\

vagrant box add G:\goad-requirements\StefanScherer_windows_2019 --name 'StefanScherer_windows_2019'
vagrant box add G:\goad-requirements\StefanScherer_windows_2016 --name 'StefanScherer_windows_2016'
vagrant box add G:\goad-requirements\bento_ubuntu-18.04 --name 'bento_ubuntu-18.04'

vagrant box list
```

Для `box` использовать имя установленного контейнера или название файла в паке с `Vargantfile`
![[Pasted image 20241027201703.png]]

Vargantfile
```ruby
Vagrant.configure("2") do |config|

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'vmware_desktop'
ENV['GOAD_VAGRANT_OPTIONS'] = 'elk'

boxes = [
  # windows server 2019
  { :name => "GOAD-DC01",  :ip => "192.168.56.10", :box => "StefanScherer_windows_2019", :os => "windows"},
  # windows server 2019
  { :name => "GOAD-DC02",  :ip => "192.168.56.11", :box => "StefanScherer_windows_2019", :os => "windows"},
  # windows server 2016
  { :name => "GOAD-DC03",  :ip => "192.168.56.12", :box => "StefanScherer_windows_2016", :os => "windows"},
  # windows server 2019
  { :name => "GOAD-SRV02", :ip => "192.168.56.22", :box => "StefanScherer_windows_2019", :os => "windows"},
  # windows server 2016
  { :name => "GOAD-SRV03", :ip => "192.168.56.23", :box => "StefanScherer_windows_2016", :os => "windows"}
]

if ENV.has_key?('GOAD_VAGRANT_OPTIONS') and ENV['GOAD_VAGRANT_OPTIONS'].include? "elk"
  boxes.append(
    { :name => "GOAD-ELK", :ip => "192.168.56.50", :box => "bento_ubuntu-18.04", :os => "linux", :forwarded_port => [
      {:guest => 22, :host => 2210, :id => "ssh"} ]
    }
  )
end

  config.vm.provider "virtualbox" do |v|
    v.memory = 4000
    v.cpus = 2
  end

  config.vm.provider "vmware_desktop" do |v|
    v.vmx["memsize"] = "4000"
    v.vmx["numvcpus"] = "2"
    v.force_vmware_license = "workstation"  # force the licence for fix some vagrant plugin issue
    v.gui = true
  end

  # disable rdp forwarded port inherited from StefanScherer box
  config.vm.network :forwarded_port, guest: 3389, host: 3389, id: "rdp", auto_correct: true, disabled: true

  # no autoupdate if vagrant-vbguest is installed
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  config.vm.boot_timeout = 600
  config.vm.graceful_halt_timeout = 600
  config.winrm.retry_limit = 30
  config.winrm.retry_delay = 10

  boxes.each do |box|
    config.vm.define box[:name] do |target|
      # BOX
      target.vm.provider "virtualbox" do |v|
        v.name = box[:name]
        v.customize ["modifyvm", :id, "--groups", "/GOAD"]
      end
      target.vm.box_download_insecure = box[:box]
      target.vm.box = box[:box]
      if box.has_key?(:box_version)
        target.vm.box_version = box[:box_version]
      end

      # issues/49
      target.vm.synced_folder '.', '/vagrant', disabled: true

      # IP
      target.vm.network :private_network, ip: box[:ip]

      # OS specific
      if box[:os] == "windows"
        target.vm.guest = :windows
        target.vm.communicator = "winrm"
        target.vm.provision :shell, :path => "../../../../vagrant/Install-WMF3Hotfix.ps1", privileged: false
        target.vm.provision :shell, :path => "../../../../vagrant/ConfigureRemotingForAnsible.ps1", privileged: false

        # fix ip for vmware
        if ENV['VAGRANT_DEFAULT_PROVIDER'] == "vmware_desktop"
          target.vm.provision :shell, :path => "../../../../vagrant/fix_ip.ps1", privileged: false, args: box[:ip]
        end

      else
        target.vm.communicator = "ssh"
      end

      if box.has_key?(:forwarded_port)
        # forwarded port explicit
        box[:forwarded_port] do |forwarded_port|
          target.vm.network :forwarded_port, guest: forwarded_port[:guest], host: forwarded_port[:host], host_ip: "127.0.0.1", id: forwarded_port[:id]
        end
      end

    end
  end
end
```

Установка или запуск
```
cd D:\GOAD\ad\GOAD\providers\vmware\
vagrant.exe up
```

