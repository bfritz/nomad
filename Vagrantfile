# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

DEFAULT_CPU_COUNT = 2
$script = <<SCRIPT
GO_VERSION="1.7.5"
INSTALL="sudo pacman -S --noconfirm --needed"


# Enable multilib repo for i386 compilation
(grep -q '^\[multilib\]' /etc/pacman.conf 2> /dev/null) \
  || (echo "\n\n[multilib]\nInclude = /etc/pacman.d/mirrorlist" | sudo tee -a /etc/pacman.conf)

sudo pacman -Sy --noconfirm


# Install base dependencies
$INSTALL base-devel git go lxc tree wget zip

# Install multilib-devel for i386 compilation.  `echo`s to respond to pacman prompts.
(pacman -Qq multilib-devel 2> /dev/null) \
  || (echo -n "\ny\ny\ny\n" | sudo pacman -S multilib-devel)

# Install cross compilers
for p in x-tools-armv7-bin; do
    (pacman -Qq $p 2> /dev/null) \
      || (
      echo "Installing $p package."
      curl https://aur.archlinux.org/cgit/aur.git/snapshot/$p.tar.gz | bsdtar -C /tmp -xf -
      cd /tmp/$p
      makepkg -si --noconfirm
    )
done

# Setup go, for development of Nomad
SRCROOT="/opt/go"
SRCPATH="/opt/gopath"

# Get the ARCH
ARCH=`uname -m | sed 's|i686|386|' | sed 's|x86_64|amd64|'`

# Install Go
if [[ $(go version) == "go version go${GO_VERSION} linux/${ARCH}" ]]; then
    echo "Go ${GO_VERSION} ${ARCH} already installed; Skipping"
else
    cd /tmp
    wget -q https://storage.googleapis.com/golang/go${GO_VERSION}.linux-${ARCH}.tar.gz
    tar -xf go${GO_VERSION}.linux-${ARCH}.tar.gz
    sudo mv go $SRCROOT
    sudo chmod 775 $SRCROOT
    sudo chown vagrant:vagrant $SRCROOT
fi

# Setup the GOPATH; even though the shared folder spec gives the working
# directory the right user/group, we need to set it properly on the
# parent path to allow subsequent "go get" commands to work.
sudo mkdir -p $SRCPATH
sudo chown -R vagrant:vagrant $SRCPATH 2>/dev/null || true
# ^^ silencing errors here because we expect this to fail for the shared folder

cat <<EOF >/tmp/gopath.sh
export GOPATH="$SRCPATH"
export PATH="$SRCROOT/bin:$SRCPATH/bin:\$PATH"
EOF
sudo mv /tmp/gopath.sh /etc/profile.d/gopath.sh
sudo chmod 0755 /etc/profile.d/gopath.sh
source /etc/profile.d/gopath.sh

# Install Docker
$INSTALL docker

# Start docker
sudo systemctl start docker

# Make sure we can actually use docker as the vagrant user
sudo usermod -aG docker vagrant

# Setup Nomad for development
cd /opt/gopath/src/github.com/hashicorp/nomad && make bootstrap

# Install rkt, consul and vault
#bash scripts/install_rkt.sh
#bash scripts/install_rkt_vagrant.sh
#bash scripts/install_consul.sh
#bash scripts/install_vault.sh

# Set hostname's IP to made advertisement Just Work
sudo sed -i -e "s/.*nomad.*/$(ip route get 1 | awk '{print $NF;exit}') $(hostname)/" /etc/hosts

# CD into the nomad working directory when we login to the VM
PROFILE=$HOME/.bash_profile
(grep "cd /opt/gopath/src/github.com/hashicorp/nomad" $PROFILE 2> /dev/null) || echo "cd /opt/gopath/src/github.com/hashicorp/nomad" >> $PROFILE
SCRIPT

def configureVM(vmCfg, vmParams={
                  numCPUs: DEFAULT_CPU_COUNT,
                }
               )
  # When updating make sure to use a box that supports VMWare and VirtualBox
  vmCfg.vm.box = "wholebits/arch-64" # Arch Linux

  vmCfg.vm.provision "shell", inline: $script, privileged: false
  vmCfg.vm.synced_folder '.', '/opt/gopath/src/github.com/hashicorp/nomad'

  # We're going to compile go and run a concurrent system, so give ourselves
  # some extra resources. Nomad will have trouble working correctly with <2
  # CPUs so we should use at least that many.
  cpus = vmParams.fetch(:numCPUs, DEFAULT_CPU_COUNT)
  memory = 2048

  vmCfg.vm.provider "parallels" do |p, o|
    p.memory = memory
    p.cpus = cpus
  end

  vmCfg.vm.provider "virtualbox" do |v|
    v.memory = memory
    v.cpus = cpus
  end

  ["vmware_fusion", "vmware_workstation"].each do |p|
    vmCfg.vm.provider p do |v|
      v.enable_vmrun_ip_lookup = false
      v.gui = false
      v.memory = memory
      v.cpus = cpus
    end
  end
  return vmCfg
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  1.upto(3) do |n|
    vmName = "nomad-server%02d" % [n]
    isFirstBox = (n == 1)

    numCPUs = DEFAULT_CPU_COUNT
    if isFirstBox and Object::RUBY_PLATFORM =~ /darwin/i
      # Override the max CPUs for the first VM
      numCPUs = [numCPUs, (`/usr/sbin/sysctl -n hw.ncpu`.to_i - 1)].max
    end

    config.vm.define vmName, autostart: isFirstBox, primary: isFirstBox do |vmCfg|
      vmCfg.vm.hostname = vmName
      vmCfg = configureVM(vmCfg, {:numCPUs => numCPUs})
    end
  end

  1.upto(3) do |n|
    vmName = "nomad-client%02d" % [n]
    config.vm.define vmName, autostart: false, primary: false do |vmCfg|
      vmCfg.vm.hostname = vmName
      vmCfg = configureVM(vmCfg)
    end
  end
end
