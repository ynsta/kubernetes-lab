# -*- mode: ruby -*-
# vi: set ft=ruby :

# --- CONFIGURATION VARIABLES ---
# We define our nodes here. If you want 5 workers later, just change "2" to "5".
WORKER_COUNT = 2
NETWORK_PREFIX = "10.4.21"
# Disk Sizes
DISK_SIZE_GB_JUMP = 20
DISK_SIZE_GB_K8S  = 40

# List your preferred keys in order of priority
possible_keys = [
  "~/.ssh/id_ecdsa.pub",
  "~/.ssh/id_ed25519.pub",
  "~/.ssh/id_rsa.pub"
]

# Find the first one that exists
selected_key_path = possible_keys.find { |path| File.exist?(File.expand_path(path)) }

if selected_key_path
  SSH_PUB_KEY = File.read(File.expand_path(selected_key_path)).strip
else
  puts "Warning: No SSH public key found!"
  SSH_PUB_KEY = ""
end

Vagrant.configure("2") do |config|
  config.vm.box = "cloud-image/debian-13"
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.ssh.forward_env = false
  config.ssh.forward_agent = true

  # --- GLOBAL PROVIDER CONFIG (LIBVIRT) ---
  config.vm.provider :libvirt do |libvirt|
    libvirt.driver = "kvm"
    libvirt.uri = 'qemu:///system'
    libvirt.cpu_mode = 'host-passthrough'
    libvirt.disk_driver :cache => 'none'
  end

  # --- COMMON PROVISIONING ---
  config.vm.provision "shell", inline: <<-SHELL
    export DEBIAN_FRONTEND=noninteractive
    apt-get update -qq
    apt-get install -y -qq curl apt-transport-https wget vim git openssl htop net-tools zsh zplug fzf tmux

    echo #{SSH_PUB_KEY} >> /home/vagrant/.ssh/authorized_keys
    echo #{SSH_PUB_KEY} >> /root/.ssh/authorized_keys

    sed -i \
      -e 's/^#*PermitRootLogin.*/PermitRootLogin prohibit-password/' \
      -e 's/^AcceptEnv.*/AcceptEnv COLORTERM NO_COLOR/' \
      /etc/ssh/sshd_config
    if systemctl is-active --quiet ssh; then
      systemctl restart ssh
    elif systemctl is-active --quiet sshd; then
      systemctl restart sshd
    fi

    echo "Configuring zsh as default for vagrant user..."
    chsh -s $(which zsh) vagrant
    cat <<"EOF" > /home/vagrant/.zshrc
# Aliases
alias k='kubectl'
alias la='ls -lah --color=auto'
alias ls='ls --color=auto'
alias grep='grep --color=auto'
alias ipb='ip --brief'

autoload -Uz promptinit
promptinit

# Use emacs keybindings even if our EDITOR is set to vi
bindkey -e

setopt histignorealldups sharehistory

# Keep 5000 lines of history within the shell and save it to ~/.zsh_history:
HISTSIZE=5000
SAVEHIST=5000
HISTFILE=~/.zsh_history

# Use modern completion system
autoload -Uz compinit
compinit

# zplug - manage plugins
source /usr/share/zplug/init.zsh

zplug "lib/*", from:oh-my-zsh
zplug "plugins/git", from:oh-my-zsh
zplug "plugins/sudo", from:oh-my-zsh
zplug "plugins/command-not-found", from:oh-my-zsh
zplug "plugins/kubectl", from:oh-my-zsh

zplug "zsh-users/zsh-syntax-highlighting"
zplug "zsh-users/zsh-autosuggestions"
zplug "zsh-users/zsh-history-substring-search"
zplug "zsh-users/zsh-completions"
zplug "junegunn/fzf"
zplug "themes/robbyrussell", from:oh-my-zsh, as:theme   # Theme

# zplug - install/load new plugins when zsh is started or reloaded
if ! zplug check; then
    zplug install
fi
zplug load

# fzf history config
source <(fzf --zsh)
EOF
    echo "Configuring tmux for vagrant user..."
    cat <<"EOF" > /home/vagrant/.tmux.conf
# Enable Mouse Support (clickable windows, resizable panes)
set -g mouse on

# Easy Config Reload (Press Prefix + r)
bind r source-file ~/.tmux.conf \; display "Reloaded!"

# Easier split commands
bind '-' split-window -h
bind '=' split-window -v
unbind '"'
unbind '%'

# Start window numbering at 1 instead of 0
set -g base-index 1
setw -g pane-base-index 1
EOF

  chown -R vagrant:vagrant /home/vagrant
  SHELL

  # --- VM 1: JUMPBOX ---
  config.vm.define "jumpbox" do |cfg|
    cfg.vm.hostname = "jumpbox"
    cfg.vm.network "private_network", ip: "#{NETWORK_PREFIX}.80"
    cfg.vm.provider :libvirt do |v|
      v.memory = 1024
      v.cpus = 1
      v.machine_virtual_size = DISK_SIZE_GB_JUMP
    end
  end

  # --- VM 2: SERVER (Control Plane) ---
  config.vm.define "server" do |cfg|
    cfg.vm.hostname = "server"
    cfg.vm.network "private_network", ip: "#{NETWORK_PREFIX}.19"
    cfg.vm.provider :libvirt do |v|
      v.memory = 4096
      v.cpus = 2
      v.machine_virtual_size = DISK_SIZE_GB_K8S
    end
  end

  # --- VM 3 & 4: WORKERS (LOOP) ---
  # Ruby loop to create node-0, node-1, node-2, etc.
  (0..WORKER_COUNT-1).each do |i|
    config.vm.define "node-#{i}" do |cfg|
      cfg.vm.hostname = "node-#{i}"
      # IP calculation: Starts at .20
      cfg.vm.network "private_network", ip: "#{NETWORK_PREFIX}.#{20 + i}"
      cfg.vm.provider :libvirt do |v|
        v.memory = 4096
        v.cpus = 2
        v.machine_virtual_size = DISK_SIZE_GB_K8S
      end
    end
  end

end
