# Generic Vagrantfile that can be used without modification in a variety of situations.
# Configuration is read from `vagrant-conf.yml` by default.

require 'rbconfig'
require 'yaml'

# FIXME: check if OS specific package manager is installed
$install_git = <<-SCRIPT
if ! command -v git >/dev/null 2>&1; then
  if command -v yum >/dev/null 2>&1; then
    # CentOS, RHEL, Oracle Linux
    yum -y install git
  elif command -v apt-get >/dev/null 2>&1; then
    # Debian, Ubuntu
    apt-get -y install git
  elif command -v dnf >/dev/null 2>&1; then
    # Fedora
    dnf -y install git
  elif command -v pacman >/dev/null 2>&1; then
    # Arch Linux
    pacman -S git
  else
    echo "package manager not yet supported"
  fi
fi
SCRIPT

# Inline plugin which makes sure a SATA controller with a given name is present in the VM
class VagrantPlugins::ProviderVirtualBox::Action::SetName
  alias_method :original_call, :call
  def call(env)
    machine = env[:machine]
    driver = machine.provider.driver
    uuid = driver.instance_eval { @uuid }
    ui = env[:ui]

    # puts "MACHINE"
    # puts env[:machine].inspect
    # puts "MACHINE.config"
    # puts env[:machine].config.inspect
    # puts "GLOBAL"
    # puts env[:global_config].inspect

    controller_name = File.read(".sata_controller.#{machine.name}")

    vm_info = driver.execute("showvminfo", uuid)
    has_this_controller = vm_info.match("Storage Controller Name.*#{controller_name}")

    if has_this_controller
      ui.info "Machine #{machine.name} already has the '#{controller_name}' HDD controller"
    else
      ui.warn "Creating SATA controller with name '#{controller_name}' on machine #{machine.name}..."
      driver.execute('storagectl', uuid,
                     '--name', "#{controller_name}",
                     '--add', 'sata',
                     '--controller', 'IntelAhci')
    end
    original_call(env)
  end
end

VAGRANTFILE_API_VERSION = '2'

# Initialize coloured script output
@ui = Vagrant::UI::Colored.new

# Set Vagrant controlfile
vagrantconf = ENV['VAGRANT_CONF'] ? ENV['VAGRANT_CONF'] : 'vagrant-conf.yml'

# Load Vagrant controlfile
vagrant_conf        = YAML::load_stream(File.open(vagrantconf))
global              = vagrant_conf[0]
hosts               = vagrant_conf[1]
inventory_groupings = vagrant_conf[2]
galaxy_requirements = vagrant_conf[3]
playbooks           = vagrant_conf[4]
# The last two entries are optional
pip_requirements    = vagrant_conf[5]
ansible_cfgs        = vagrant_conf[6]

pip_requirements = nil if pip_requirements == {}
ansible_cfgs     = nil if ansible_cfgs == {}

# If no hostname is specified in the configuration file use the name `ansiblehost`
if hosts.empty?
  hosts = [{"hostname"=>"ansiblehost"}]
end

# {{{ Helper functions

def windows_host?
  Vagrant::Util::Platform.windows?
end

def cygwin_host?
  Vagrant::Util::Platform.cygwin?
end

def run_locally?
  # Pretend that Cygwin is NOT Windows, it's POSIX
  windows_host? && !cygwin_host? || FORCE_LOCAL_RUN
end

def set_global_default(global, param, default)
  global.key?(param) ? global[param] : default
end

def set_host_default(global, host, param, default)
  host.key?(param) ? host[param] : (global.key?(param) ? global[param] : default)
end

def get_global_parameter(global, param)
  global[param] if global.key?(param)
end

def get_config_parameter(host, global, param)
  host.key?(param) ? host[param] : get_global_parameter(global, param)
end

# Convert an array of hashes into a single hash
def array_of_hashes_2_hash(arr)
  Hash[*arr.collect{|h| h.to_a}.flatten]
end

# Merge two array of hashes
def merge_2_array_of_hashes(a_arr, b_arr)
  merge_hash = a_hash = array_of_hashes_2_hash(a_arr) if a_arr
  merge_hash = b_hash = array_of_hashes_2_hash(b_arr) if b_arr
  merge_hash = a_hash.merge!(b_hash)                  if (a_arr and b_arr)
  merge_hash
end

# }}}

# {{{ Initialize global environment from Vagrant configuration or use sensible defaults

# By default, Vagrant expects a "vagrant" user to SSH into the machine as. This
# user should be setup with the insecure keypair that Vagrant uses as a default
# to attempt to SSH.
VAGRANT_USER                = set_global_default(global, 'VAGRANT_USER',                'vagrant')
# Vagrant provisioner support
RUN_FILE_PROVISIONER        = set_global_default(global, 'RUN_FILE_PROVISIONER',        true)
RUN_SHELL_PROVISIONER       = set_global_default(global, 'RUN_SHELL_PROVISIONER',       true)
RUN_ANSIBLE_PROVISIONER     = set_global_default(global, 'RUN_ANSIBLE_PROVISIONER',     true)
# Set default LC_ALL for all VM boxes
ENV["LC_ALL"]               = set_global_default(global, 'LC_ALL',                      'en_US.UTF-8')
# When set to `true`, Ansible will be forced to be run locally on the VM
# instead of from the host machine (provided Ansible is installed).
FORCE_LOCAL_RUN             = set_global_default(global, 'FORCE_LOCAL_RUN',             false)
# Vagrant Proxy Plugin Configuration
USE_PROXY                   = set_global_default(global, 'USE_PROXY',                   false)
# Vagrant Hostmanager Plugin Configuration
USE_HOSTMANAGER             = set_global_default(global, 'USE_HOSTMANAGER',             true)
# The name of the Ansible Galaxy role file
ANSIBLE_GALAXY_ROLE_FILE    = set_global_default(global, 'ANSIBLE_GALAXY_ROLE_FILE',    '.requirements.yml')
# Should `ansible-galaxy` overwrite roles after initial download
ANSIBLE_GALAXY_OVERWRITE    = set_global_default(global, 'ANSIBLE_GALAXY_OVERWRITE',    true)
ANSIBLE_INSTALL             = set_global_default(global, 'ANSIBLE_INSTALL',             true)
ANSIBLE_INSTALL_MODE        = set_global_default(global, 'ANSIBLE_INSTALL_MODE',        'default')
ANSIBLE_VERSION             = set_global_default(global, 'ANSIBLE_VERSION',             'latest')
ANSIBLE_PIP_ARGS            = set_global_default(global, 'ANSIBLE_PIP_ARGS',            nil)
ANSIBLE_PIP_REQUIREMENTS    = set_global_default(global, 'ANSIBLE_PIP_REQUIREMENTS',    '.requirements.txt')
# Ansible's verbosity to obtain detailed logging (false/true/vvv/vvvv)
ANSIBLE_VERBOSE             = set_global_default(global, 'ANSIBLE_VERBOSE',             false)
ANSIBLE_FOLDER              = set_global_default(global, 'ANSIBLE_FOLDER',              'ansible/')
ANSIBLE_BECOME              = set_global_default(global, 'ANSIBLE_BECOME',              true)
ANSIBLE_BECOME_USER         = set_global_default(global, 'ANSIBLE_BECOME_USER',         'root')
ANSIBLE_PLAYBOOK_PREFIX     = set_global_default(global, 'ANSIBLE_PLAYBOOK_PREFIX',     'vagrant-')
# Ansible playbook. It must be located in the ansible folder of the project.
ANSIBLE_PLAYBOOK            = set_global_default(global, 'ANSIBLE_PLAYBOOK',            'site.yml')
ANSIBLE_LIMIT               = set_global_default(global, 'ANSIBLE_LIMIT',               'all')
ANSIBLE_CFG                 = set_global_default(global, 'ANSIBLE_CFG',                 'ansible.cfg')
ANSIBLE_LOCAL_CONFIG        = set_global_default(global, 'ANSIBLE_LOCAL_CONFIG',        '.ansible-local.cfg')
ANSIBLE_CONFIG              = set_global_default(global, 'ANSIBLE_CONFIG',              '.ansible.cfg')
ANSIBLE_LOCAL_INVENTORY     = set_global_default(global, 'ANSIBLE_LOCAL_INVENTORY',     '.vagrant-local-inventory.ini')
ANSIBLE_INVENTORY           = set_global_default(global, 'ANSIBLE_INVENTORY',           '.vagrant-inventory.ini')
ANSIBLE_RAW_ARGS            = set_global_default(global, 'ANSIBLE_RAW_ARGS',            nil)
ANSIBLE_VAULT_PASSWORD_FILE = set_global_default(global, 'ANSIBLE_VAULT_PASSWORD_FILE', nil)
# If true, X11 forwarding over SSH connections is enabled.
SSH_FORWARD_X11             = set_global_default(global, 'SSH_FORWARD_X11',             true)

# }}}

# {{{ Make sure important plugins are installed

def install_plugins
  unless Vagrant.has_plugin?("vagrant-vbguest")
    @ui.info("Installing vagrant-vbguest Plugin...")
    system('vagrant plugin install vagrant-vbguest')
  end

  if USE_PROXY
    unless Vagrant.has_plugin?("vagrant-proxyconf")
      @ui.info("Installing vagrant-proxyconf Plugin...")
      system('vagrant plugin install vagrant-proxyconf')
    end
  end

  if USE_HOSTMANAGER
    unless Vagrant.has_plugin?("vagrant-hostmanager")
      @ui.info("Installing vagrant-hostmanager Plugin...")
      system('vagrant plugin install vagrant-hostmanager')
    end
  end
end

# }}}

# {{{ Configure plugins

def configure_proxy_plugin
  # Add optional proxy configuration from host environment
  if USE_PROXY
    if Vagrant.has_plugin?("vagrant-proxyconf")
      @ui.warn("Getting Proxy Configuration from Host...")
      if ENV["http_proxy"]
        @ui.info("http_proxy: " + ENV["http_proxy"])
        config.proxy.http = ENV["http_proxy"]
      end
      if ENV["https_proxy"]
        @ui.info("https_proxy: " + ENV["https_proxy"])
        config.proxy.https = ENV["https_proxy"]
      end
      if ENV["no_proxy"]
        @ui.info("no_proxy: " + ENV["no_proxy"])
        config.proxy.no_proxy = ENV["no_proxy"]
      end
    end
  end
end

# }}}

# {{{ Generate the Ansible Galaxy requirements file

def generate_ansible_galaxy_requirements(galaxy_requirements)
  # Generate the requirements YAML file for ansible-galaxy (use just linefeeds as EOL on Cygwin instead of CLRF, 'wb')
  File.open(ANSIBLE_FOLDER + ANSIBLE_GALAXY_ROLE_FILE, "wb") { |file| file.write(galaxy_requirements.to_yaml) }
end

# }}}

# {{{ Generate the requirements file for pip install

def generate_pip_install_requirements(pip_requirements)
  if pip_requirements.nil?
    pip_requirements = <<-FILE
-e  git+https://github.com/ansible/ansible.git#egg=devel
FILE
    File.open(ANSIBLE_PIP_REQUIREMENTS, "wb") { |file| file.write(pip_requirements) }
  else
    File.open(ANSIBLE_PIP_REQUIREMENTS, "wb") { |file| file.write(pip_requirements.values.join) }
  end
end

# }}}

# {{{ Generate the Ansible configuration files

def generate_ansible_configfiles(ansible_cfgs)
  if ansible_cfgs.nil?
    # In case there is NO config information...
    ansible_cfg = <<-FILE
[defaults]
# Disable SSH key host checking
host_key_checking = no
# Additional paths to search for roles, colon separated
roles_path = ./ansible/roles

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o UserKnownHostsFile=/dev/null -o IdentitiesOnly=yes
FILE
    # Otherwise,... generate the Ansible configuration file for `ansible_local` mode provisioner
    File.open(ANSIBLE_FOLDER + ANSIBLE_LOCAL_CONFIG, "wb") { |file| file.write(ansible_cfg) }

    ansible_cfg=  <<-FILE
[defaults]
# Disable SSH key host checking
host_key_checking = no
# Additional paths to search for roles, colon separated
roles_path = ./ansible/roles

[ssh_connection]
# ssh arguments to use (`ControlMaster=no` is needed for Ansible to work on Cygwin).
# Needs to be explicitly set by Vagrant with ansible.raw_ssh_args in Vagrantfile.
# Vagrant uses ANSIBLE_SSH_ARGS for Cygwin which has higher precedence than Ansible configuration options.
ssh_args = -o ControlMaster=no
FILE
    # Otherwise,... generate the Ansible configuration file for `ansible` mode provisioner
    File.open(ANSIBLE_FOLDER + ANSIBLE_CONFIG, "wb") { |file| file.write(ansible_cfg) }
  else
    # Generate the Ansible configuration file for `ansible_local` mode provisioner
    File.open(ANSIBLE_LOCAL_CONFIG, "wb") { |file| file.write(ansible_cfgs[ANSIBLE_LOCAL_CONFIG]) }
    # Generate the Ansible configuration file for `ansible mode` provisioner
    File.open(ANSIBLE_CONFIG, "wb") { |file| file.write(ansible_cfgs[ANSIBLE_CONFIG]) }
  end

  # Generate the Ansible configuration file for calling `ansible-playbook` manually
  ansible_cfg = <<-FILE
[defaults]
# Disable SSH key host checking
host_key_checking = no
# Additional paths to search for roles, colon separated
roles_path = ./ansible/roles

[ssh_connection]
# ssh arguments to use (`ControlMaster=no` is needed for Ansible to work on Cygwin).
ssh_args = -o ControlMaster=no
# ssh arguments to use on Unix systems
#ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o UserKnownHostsFile=/dev/null -o IdentitiesOnly=yes
FILE
    File.open(ANSIBLE_FOLDER + ANSIBLE_CFG, "wb") { |file| file.write(ansible_cfg) } if !File.file?(ANSIBLE_FOLDER + ANSIBLE_CFG)
end

# }}}

# {{{ Generate the Ansible playbook(s)

def generate_ansible_playbooks(hosts, galaxy_requirements, playbooks)
  if playbooks.empty?
    # Generate the Ansible playbook with an empty YAML array
    File.open(ANSIBLE_FOLDER + '.' + ANSIBLE_PLAYBOOK_PREFIX + ANSIBLE_PLAYBOOK, "wb") { |file| file.write('--- []') }
    # Generate a skeleton Ansible playbook based on the contents of the `ansible-galaxy` requirements file
    File.open(ANSIBLE_FOLDER + ANSIBLE_PLAYBOOK, "ab") { |file|
      file << "---\n"
      hosts.each do |host|
        file << "- hosts: #{host['hostname']}\n"
        file << "  become: true\n"
        file << "  roles:\n"
        galaxy_requirements.each do |role|
          file << "    - { role: #{role['name']},  tags: #{role['name']} }\n"
        end
        file << "\n"
      end
      file << "..."
    } if !File.file?(ANSIBLE_FOLDER + ANSIBLE_PLAYBOOK)
  else
    playbooks.each do |key, value|
      vagrant_playbook = ANSIBLE_FOLDER + '.' + ANSIBLE_PLAYBOOK_PREFIX + key 
      playbook         = ANSIBLE_FOLDER + key 
      # Generate the Ansible playbook(s) for the Vagrant Ansible provisioner
      File.open(vagrant_playbook, "wb") { |file| file.write(value) }
      # Generate the Ansible playbook(s) for calling `ansible-playbook` manually if they do not exist
      File.open(playbook, "wb") { |file| file.write(value) } if !File.file?(playbook)
    end
  end
end

# }}}

# {{{ Generate Ansible inventory files

def generate_ansible_local_inventory(global, hosts, inventory_groupings)
  i = 0
  # Generate inventory file for ansible local provisioner
  File.open(ANSIBLE_FOLDER + ANSIBLE_LOCAL_INVENTORY, 'wb') { |file|
    hosts.each do |host|
      i = i + 1
      python_interpreter = get_config_parameter(host, global, 'python_interpreter')
      if i == hosts.length
        if python_interpreter
          file.write("#{host['hostname']} ansible_connection=local ansible_python_interpreter=#{python_interpreter}\n")
        else
          file.write("#{host['hostname']} ansible_connection=local\n")
        end
      else
        if python_interpreter
          file.write("#{host['hostname']} ansible_ssh_user=#{VAGRANT_USER} ansible_ssh_private_key_file=/vagrant/.vagrant/insecure_private_key ansible_python_interpreter=#{python_interpreter}\n")
        else
          file.write("#{host['hostname']} ansible_ssh_user=#{VAGRANT_USER} ansible_ssh_private_key_file=/vagrant/.vagrant/insecure_private_key\n")
        end
      end
    end
  }
  # Append inventory groupings
  File.open(ANSIBLE_FOLDER + ANSIBLE_LOCAL_INVENTORY, 'ab') { |file| file.write(inventory_groupings.values.join) } if !inventory_groupings.empty?
end

def generate_ansible_inventory(global, hosts, inventory_groupings)
  i = 0
  # Generate inventory file for ansible provisioner (use just linefeeds as EOL on Cygwin, 'wb')
  File.open(ANSIBLE_FOLDER + ANSIBLE_INVENTORY, 'wb') { |file|
    hosts.each do |host|
      i = i + 1
      python_interpreter = get_config_parameter(host, global, 'python_interpreter')
      if host.has_key?('forwarded_ssh_port')
        ansible_ssh_port = host['forwarded_ssh_port']['host']
      else
        if host.has_key?('private_networks') && host['private_networks'][0]['ip'] != 'dhcp'
          ansible_ssh_port = 11000 + /^(\d{1,3}).(\d{1,3}).(\d{1,3}).(\d{1,3})$/.match(host['private_networks'][0]['ip'])[4].to_i
        else
          ansible_ssh_port = 11000 + i
        end
      end
      if python_interpreter
        file.write("#{host['hostname']} ansible_ssh_host=127.0.0.1 ansible_ssh_port=#{ansible_ssh_port} ansible_ssh_user=#{VAGRANT_USER} ansible_ssh_private_key_file='../.vagrant/insecure_private_key' ansible_python_interpreter=#{python_interpreter}\n")
      else
        file.write("#{host['hostname']} ansible_ssh_host=127.0.0.1 ansible_ssh_port=#{ansible_ssh_port} ansible_ssh_user=#{VAGRANT_USER} ansible_ssh_private_key_file='../.vagrant/insecure_private_key'\n")
      end
    end
  }
  # Append localhost
  File.open(ANSIBLE_FOLDER + ANSIBLE_INVENTORY, 'ab') { |file| file.write("127.0.0.1 ansible_connection=local\n") }
  # Append inventory groupings
  File.open(ANSIBLE_FOLDER + ANSIBLE_INVENTORY, 'ab') { |file| file.write(inventory_groupings.values.join) } if !inventory_groupings.empty?
end

def generate_ansible_inventories(global, hosts, inventory_groupings)
  generate_ansible_local_inventory(global, hosts, inventory_groupings)
  generate_ansible_inventory(global, hosts, inventory_groupings)
  if !File.file?('./ansible/inventory.ini')
    run_locally? ? system('cp ./ansible/.vagrant-local-inventory.ini ./ansible/inventory.ini')
    : system('cp ./ansible/.vagrant-inventory.ini ./ansible/inventory.ini')
  end
end

# }}}

# {{{ Provisioning functions

def provision_ansible(vm, host, global)
  if run_locally?
    ansible_mode = 'ansible_local'
  else
    ansible_mode = 'ansible'
  end

  # In "ansible_local" mode we have to make sure "git" is installed
  # The requirements section in our Vagrant configuration file for ansible-galaxy could use Git/Bitbucket URLs
  if ansible_mode == 'ansible_local'
    vm.provision :shell, name: "Making sure Git is installed on Ansible controlhost...", inline: $install_git
  end

  # Provisioning configuration for Ansible
  vm.provision ansible_mode do |ansible|
    if !ANSIBLE_GALAXY_OVERWRITE
      ansible.galaxy_command = "ansible-galaxy install --role-file=%{role_file} --roles-path=%{roles_path}"
    end
    ansible.compatibility_mode  = "2.0"
    ansible.version             = ANSIBLE_VERSION
    ansible.verbose             = ANSIBLE_VERBOSE
    ansible.become              = ANSIBLE_BECOME
    ansible.become_user         = ANSIBLE_BECOME_USER
    # Disable default limit to connect to all the machines
    ansible.limit               = ANSIBLE_LIMIT
    ansible.tags                = get_global_parameter(global, 'ANSIBLE_TAGS')
    ansible.skip_tags           = get_global_parameter(global, 'ANSIBLE_SKIP_TAGS')
    # This does not currently work when task names have blanks in their name on the Cygwin platform
    ansible.start_at_task       = get_global_parameter(global, 'ANSIBLE_START_AT_TASK')
    ansible.vault_password_file = ANSIBLE_VAULT_PASSWORD_FILE
    if ansible_mode == 'ansible' && cygwin_host?
      ansible.raw_arguments     = ANSIBLE_RAW_ARGS
      ansible.raw_ssh_args      = ['-o ControlMaster=no']
      ansible.galaxy_role_file  = ANSIBLE_FOLDER + ANSIBLE_GALAXY_ROLE_FILE
      ansible.config_file       = ANSIBLE_FOLDER + ANSIBLE_CONFIG
      ansible.inventory_path    = ANSIBLE_FOLDER + ANSIBLE_INVENTORY
      ansible.playbook          = ANSIBLE_FOLDER + '.' + ANSIBLE_PLAYBOOK_PREFIX + ANSIBLE_PLAYBOOK
    end
    if ansible_mode == 'ansible_local'
      ansible.install           = ANSIBLE_INSTALL
      ansible.install_mode      = ANSIBLE_INSTALL_MODE
      ansible.pip_args          = ANSIBLE_PIP_ARGS
      ansible.tmp_path          = "/tmp/vagrant-ansible"
      ansible.provisioning_path = "/vagrant/ansible"
      ansible.galaxy_role_file  = ANSIBLE_GALAXY_ROLE_FILE
      ansible.config_file       = ANSIBLE_LOCAL_CONFIG
      ansible.inventory_path    = ANSIBLE_LOCAL_INVENTORY
      ansible.playbook          = '.' + ANSIBLE_PLAYBOOK_PREFIX + ANSIBLE_PLAYBOOK
    end
  end
end

# Vagrant private networks allow you to access your guest machine by some address that is not publicly
# accessible from the global internet. In general, this means your machine gets an address in the private address space.
# Multiple machines within the same private network (also usually with the restriction that they're backed by the same provider)
# can communicate with each other on private networks. By default, private networks are host-only networks.
# The Vagrant VirtualBox provider supports using the private network as a VirtualBox internal network.
# Options:
#   ip          - Either an IP/IPv6 address or the string `dhcp`. (optional, default `dhcp`)
#   netmask     - The network mask. (optional, default `255.255.255.0`)
#   mac         - The MAC adress attached to the network device (optional)
#   auto_config - Vagrant will not configure this network interface if set to `false` (optional, default `true`)
#   intnet      - Either `true` for the default internal network, or specify the name of the internal network (optional)
# Example:
#   private_networks:
#     - ip: dhcp
#     - ip: 192.168.56.110
#     - ip: 10.2.2.10
#       intnet: true
#     - ip: 10.0.19.10
#       intnet: myintnet
def private_networks(vm, host)
  if host.has_key?('private_networks')
    private_networks = host['private_networks']
    private_networks.each do |private_network|
      options = {}
      if private_network.key?('ip') && private_network['ip'] != 'dhcp'
        options[:ip] = private_network['ip']
        options[:netmask] = private_network['netmask'] ||= '255.255.255.0'
      else
        options[:type]    = 'dhcp'
      end
      options[:mac]                = private_network['mac'].gsub(/[-:]/, '') if private_network.key?('mac')
      options[:auto_config]        = private_network['auto_config']          if private_network.key?('auto_config')
      options[:virtualbox__intnet] = private_network['intnet']               if private_network.key?('intnet')
      vm.network :private_network, options
    end
  end
end

# Vagrant public networks are less private than private networks, and the exact meaning
# actually varies from provider to provider, hence the ambiguous definition. The idea is
# that while private networks should never allow the general public access to your machine, public networks can.
# The Vagrant VirtualBox provider supports using the public network as a VirtualBox `bridged network`.
# Options:
#   ip          - Either an IP/IPv6 address or the string `dhcp`.
#   auto_config - Vagrant will not configure this network interface if set to `false` (optional, default `true`)
# Example:
#   public_networks:
#     - ip: 192.168.2.90
def public_networks(vm, host)
  if host.has_key?('public_networks')
    public_networks = host['public_networks']
    public_networks.each do |public_network|
      options = {}
      options[:ip]          = public_network['ip']          if public_network.key?('ip') && public_network['ip'] != 'dhcp'
      options[:auto_config] = public_network['auto_config'] if public_network.key?('auto_config')
      vm.network :public_network, options
    end
  end
end

# Vagrant forwarded ports allow you to access a port on your host machine and have all data forwarded
# to a port on the guest machine, over either TCP or UDP.
# Options:
#   guest (int) - The port on the guest that you want to be exposed on the host. This can be any port.
#   host (int)  - The port on the host that you want to use to access the port on the guest.
#                 This must be greater than port 1024 unless Vagrant is running as root (which is not recommended).
# Example:
#   forwarded_ports:
#     - { guest: 80, host: 8081 }
#     - { guest: 90, host: 9091 }
def forwarded_ports(vm, host)
  if host.has_key?('forwarded_ports')
    ports = host['forwarded_ports']
    ports.each do |port|
      vm.network "forwarded_port", guest: port['guest'], host: port['host']
    end
  end
end

# Vagrant uses this entry for SSH access into the guest machine instead of the
# default port 2222 for the first machine and an arbitrary port for the rest
# of the machines to avoid conflicts.
# Options:
#   guest (int) - The port on the guest where the SSH daemon is listening. (optional, defaults to `22`)
#   host (int)  - The port on the host that you want to use to access the port on the guest. (optional, see comment above)
#                 This must be greater than port 1024 unless Vagrant is running as root (which is not recommended).
# Example:
#   forwarded_ssh_port:
#     host: 11110
def forwarded_ssh_port(vm, host, i)
  if host.has_key?('forwarded_ssh_port')
    ssh_port = host['forwarded_ssh_port']
    vm.network "forwarded_port", guest: ssh_port.has_key?('guest') ? ssh_port['guest'] : 22, host: ssh_port['host'], id: "ssh"
  else
    if host.has_key?('private_networks') && host['private_networks'][0]['ip'] != 'dhcp'
      last_part = /^(\d{1,3}).(\d{1,3}).(\d{1,3}).(\d{1,3})$/.match(host['private_networks'][0]['ip'])[4].to_i
    else
      last_part = i
    end
    vm.network "forwarded_port", guest: 22, host: 11000 + last_part , id: "ssh"
  end
end

# Synced folders enable Vagrant to sync a folder on the host machine to the guest machine.
# By default, Vagrant mounts the synced folders with the owner/group set to the SSH user and any parent folders set to root.
# By default, Vagrant will share your project directory (the directory with the Vagrantfile) to `/vagrant`.
# Options:
#   src (string)    - Path to a directory on the host machine. If the path is relative, it is relative to the project root.
#   dest: (string)  - Absolute path of where to share the folder within the guest machine.
#                     This folder will be created (recursively, if it must) if it does not exist.
#   options: (hash) - See the documentation at https://www.vagrantup.com/docs/synced-folders/
#                     what you can use for the `options` clause. (optional)
#     :type: (string)         - The type of synced folder. If this is not specified, Vagrant will automatically
#                               choose the best synced folder option for your environment.
#     :create: (boolean)      - If true, the host path will be created if it does not exist. Defaults to `false`.
#     :disabled: (boolean)    - If true, this synced folder will be disabled and will not be setup. Defaults to `false`.
#     :owner: (string)        - The user who should be the owner of this synced folder. By default this will be the SSH user.
#     :group: (string)        - The group that will own the synced folder. By default this will be the SSH user.
#     :id: (string)           - The name for the mount point of this synced folder in the guest machine.
#                               This shows up when you run mount in the guest machine.
#     :mount_options: (array) - A list of additional mount options to pass to the mount command.
# There are also `synced folder type` specific options. All suboptions must start with a `:`, eg. `:type: virtualbox`.
# Example:
#   synced_folders:
#     - src: "D:/temp/netbox"
#       dest: "/media/netbox"
#       options:
#         :type: rsync
#     - src: "."
#       dest: "/vagrant"
#       options:
#         :type: virtualbox
#         :mount_options: ['dmode=0700', 'fmode=0600']
#     - src: "C:/temp/books"
#       dest: "/books"
#       options:
#         :owner: "vagrant",
#         :group: "vagrant",
#         :mount_options: ['uid=1234', 'gid=1234']
def synced_folders(vm, host, global)
  if folders = get_config_parameter(host, global, 'synced_folders')
    folders.each do |folder|
      vm.synced_folder folder['src'], folder['dest'], folder['options']
    end
  end
end

# The Vagrant file provisioner allows you to upload a file or directory from the host machine to the guest machine.
# The file/folder is uploaded as the SSH user over SCP, so this location must be writable to that user.
# Options:
#   source (string)      - Is the local path of the file or directory to be uploaded.
#   destination (string) - Is the remote path on the guest machine where the source will be uploaded to.
# Example:
#   file_provisioning:
#    - source: "~/.gitconfig"
#      destination: ".gitconfig"
#    - source: "/path/to/host/folder"
#      destination: "$HOME/remote/newfolder"
def file_provisioners(vm, host, global)
  if fps = get_config_parameter(host, global, 'file_provisioning')
    fps.each do |fp|
      vm.provision :file, source: fp['source'], destination: fp['destination']
    end
  end
end

# The Vagrant Shell provisioner allows you to upload and execute a script within the guest machine.
# An `inline` script is a script that is given to Vagrant directly within the Vagrantfile.
# An `external` script is uploaded either from the host filesystem or you can pass in its URL.
# To run a script already available on the guest you can use an inline script to invoke the remote script on the guest.
# For POSIX-like machines, the shell provisioner executes scripts with SSH.
# By default, provisioners are only run once, during the first `vagrant up` since the last vagrant destroy,
# unless the `--provision` flag is set. Optionally, you can configure provisioners to run on every up or reload.
# They will only be not run if the `--no-provision` flag is explicitly specified.
# There are six different combinations of shell provisioners which can be configured. All accept a list of shell scripts.
#   inline_shell:         - Run only once
#   inline_shell_always:  - Run on every `up` or `reload`
#   inline_shell_never:   - If you have an optional provisioner that you want to mention to the user in a "post up message"
#                           or that requires some other configuration before it is possible,
#                           then call this with `vagrant provision --provision-with <name>`.
#   scripts:              - Run only once
#   scripts_always:       - Run on every `up` or `reload`
#   scripts_never:        - If you have an optional provisioner that you want to mention to the user in a "post up message"
#                           or that requires some other configuration before it is possible,
#                           then call this with `vagrant provision --provision-with <name>`.
# Options:
#   name: (string)   - This value will be displayed in the output so that identification by the user
#                      is easier when many shell provisioners are present. (optional, except for `*_never`-type provisioners)
#   inline: (string) - Specifies the shell command(s) inline to execute on the remote machine.
#                      You can use the YAML multiline string syntax `inline: |` for lots of lines of text.  
#   script: (string) - Path to a shell script to upload and execute. It can be a script relative
#                      to the project Vagrantfile or a remote script (like a gist URL).
#   args: (string or array) - Arguments to pass to the shell script when executing it as a single string. (optional)
#                             These arguments must be written as if they were typed directly on the command
#                             line, so be sure to escape characters, quote, etc. as needed.
#                             You may also pass the arguments in using an array. In this case, Vagrant will handle quoting for you.
#  One of `inline` or `script` is required depending on the shell provisioner type.
# Example:
#   inline_shell:
#     - name: "INLINE_SHELL: Hello World inline script (just once)"
#       inline: |
#         echo This one is $1
#         echo This two is $2
#         for i in 1 2 3; do
#           echo i = $i
#         done
#       args:
#         - "hello, \ world \\ check the . and file-URL file://xyz/dir + \"string\"  + 'string' + `uname -a` !"
#         - "--param-name + $1"
#   inline_shell_always:
#     - inline: "/bin/sh /path/to/the/script/already/on/the/guest.sh"
#   inline_shell_never:
#     - name: "bootstrap-inline"
#       inline: "echo bootstrap"
#   scripts:
#     - script: "https://example.com/provisioner.sh"
#   scripts_always:
#     - script: "./scripts/test1.sh"
#     - name: Test2
#       script: "./scripts/test2.sh"
#       args: "'hello, world' p2 p3"
#     - name: Test2
#       script: "./scripts/test2.sh"
#       args:
#         - "hello, world"
#         - p2
#         - p3
#   scripts_never:
#     - name: "bootstrap-script"
#       script: "./scripts/test1.sh"

# Script based shell provisioners to be run just once
def shell_provisioners(vm, host, global)
  if scripts = get_config_parameter(host, global, 'scripts')
    scripts.each do |script|
      vm.provision :shell do |s|
        s.path = script['script']
        s.args = script['args']
        s.name = script['name']
      end
    end
  end
end

# Script based shell provisioners to be run on every `up` or `reload`
def shell_provisioners_always(vm, host, global)
  if scripts = get_config_parameter(host, global, 'scripts_always')
    scripts.each do |script|
      vm.provision :shell, run: "always" do |s|
        s.path = script['script']
        s.args = script['args']
        s.name = script['name']
      end
    end
  end
end

# Script based shell provisioners to be run never
def shell_provisioners_never(vm, host, global)
  if scripts = get_config_parameter(host, global, 'scripts_never')
    scripts.each do |script|
      vm.provision script['name'], type: "shell", run: "never" do |s|
        s.path = script['script']
        s.args = script['args']
      end
    end
  end
end

# Inline shell provisioners to be run just once
def inline_shell_provisioners(vm, host, global)
  if scripts = get_config_parameter(host, global, 'inline_shell')
    scripts.each do |script|
      vm.provision :shell do |s|
        s.inline = script['inline']
        s.args   = script['args']
        s.name   = script['name']
      end
    end
  end
end

# Inline shell provisioners to be run on every `up` or `reload`
def inline_shell_provisioners_always(vm, host, global)
  if scripts = get_config_parameter(host, global, 'inline_shell_always')
    scripts.each do |script|
      vm.provision :shell, run: "always" do |s|
        s.inline = script['inline']
        s.args   = script['args']
        s.name   = script['name']
      end
    end
  end
end

# Inline shell provisioners to be run never
def inline_shell_provisioners_never(vm, host, global)
  if scripts = get_config_parameter(host, global, 'inline_shell_never')
    scripts.each do |script|
      vm.provision script['name'], type: "shell", run: "never" do |s|
        s.inline = script['inline']
        s.args   = script['args']
      end
    end
  end
end

# Change the properties of a registered virtual machine which is not running by merging
# options from the global and VM specific section.
def merge_vm_parameters(host, global, vb)
  # These are arrays of hashes
  if global['vm_options'] or host['vm_options']
    merge_hash = merge_2_array_of_hashes(global['vm_options'], host['vm_options'])
    merge_hash.each do |key, value|
      vb.customize ["modifyvm", :id, "--#{key}", value]
    end
  end
end

# Create new Standard disks and attach them to a SATA controller.
# Disk information is merged from the global and VM specific section.
def merge_vm_disks(host, global, vb, sata_controller)
  vb_dir="./.virtualbox/"
  if global['vm_disks'] or host['vm_disks']
    merge_hash = merge_2_array_of_hashes(global['vm_disks'], host['vm_disks'])
    merge_hash.each do |key, value|
      diskname="#{vb_dir}#{host['hostname']}-#{key}.vdi"
      unless File.exist?(diskname)
        vb.customize ["createmedium", "disk", "--filename", diskname, "--size", value * 1024 , "--format", "vdi", "--variant", "Standard"]
      end
      vb.customize ["storageattach", :id , "--storagectl", sata_controller, "--port", key, "--device", "0", "--type", "hdd", "--medium", diskname]
    end
  end
end

# }}}

install_plugins()
if RUN_ANSIBLE_PROVISIONER
  generate_pip_install_requirements(pip_requirements)
  generate_ansible_galaxy_requirements(galaxy_requirements)
  generate_ansible_configfiles(ansible_cfgs)
  generate_ansible_inventories(global, hosts, inventory_groupings)
  generate_ansible_playbooks(hosts, galaxy_requirements, playbooks)
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # TODO: Implement trigger functionality
  # config.trigger.before :up do |trigger|
  #   trigger.name = "Needs to be evaluated"
  #   trigger.info = "I am running BEFORE vagrant up!!"
  # end

  # config.trigger.after :up do |trigger|
  #   trigger.name = "Needs to be evaluated"
  #   trigger.info = "I am running AFTER vagrant up!!"
  # end

  # config.trigger.before :all do |trigger|
  #   trigger.name = "Needs to be evaluated"
  #   trigger.info = "I am running BEFORE EVERY vagrant command!!"
  # end

  if Vagrant.version?(">= 2.3.0")
    config.trigger.before :status, type: :command do |t|
      t.info = "Before STATUS command!!!!!!!"
    end
  end

  # Add optional proxy configuration from host environment
  if USE_PROXY
    configure_proxy_plugin()
  else
    config.proxy.enabled = false
  end

  # Configure hostmanager plugin
  if USE_HOSTMANAGER
    config.hostmanager.enabled      = true
    config.hostmanager.manage_guest = true
  end

  # Vagrant 1.7+ automatically inserts a different insecure keypair for each new
  # VM created. The easiest way to use the same keypair for all the machines is
  # to disable this feature and rely on the legacy insecure key.
  config.ssh.insert_key  = false
  config.ssh.forward_x11 = SSH_FORWARD_X11

  i = 0
  hosts.each do |host|
    config.vm.define host['hostname'] do |node|
      i = i + 1

      # Initialize host specific environment from Vagrant configuration or use sensible defaults
      controlhost            = set_host_default(global, host, 'controlhost',            false)
      box                    = set_host_default(global, host, 'box',                    'centos/7')
      box_url                = get_config_parameter(host, global, 'box_url')
      box_version            = get_config_parameter(host, global, 'box_version')
      box_check_update       = set_host_default(global, host, 'box_check_update',       true)
      box_vagrantfile_ignore = set_host_default(global, host, 'box_vagrantfile_ignore', true)
      vb_guest_auto_update   = set_host_default(global, host, 'vb_guest_auto_update',   true)
      sata_controller        = set_host_default(global, host, 'sata_controller',        'SATA Controller')
      vm_gui                 = set_host_default(global, host, 'vm_gui',                 false)
      vm_auto_nat_dns_proxy  = set_host_default(global, host, 'vm_auto_nat_dns_proxy',  true)
      vm_name                = set_host_default(global, host, 'vm_name',                host['hostname'])
      vm_groups              = set_host_default(global, host, 'vm_groups',              '/' + File.basename(Dir.getwd))
      vm_memory              = set_host_default(global, host, 'vm_memory',              1024)
      vm_cpus                = set_host_default(global, host, 'vm_cpus',                1)
      hostname               = host.key?('domain') ? host['hostname'] + '.' + host['domain'] :
                                 (global.key?('domain') ? host['hostname'] + '.' + global['domain'] : host['hostname'])

      # Generate the files with the SATA Controller name for each box
      # An inline plugin makes sure a SATA controller with a given name is present in the VM
      File.open(".sata_controller.#{host['hostname']}", "wb") { |file| file.write(sata_controller) }
      
      if Vagrant.has_plugin?("vagrant-vbguest")
        node.vbguest.auto_update = vb_guest_auto_update
      end

      # Vagrant box information
      node.vm.box                    = box
      node.vm.box_url                = box_url
      node.vm.box_version            = box_version
      node.vm.box_check_update       = box_check_update
      node.vm.ignore_box_vagrantfile = box_vagrantfile_ignore

      # Network setup
      node.vm.hostname = hostname
      private_networks(node.vm, host)
      public_networks(node.vm, host)
      forwarded_ssh_port(node.vm, host, i)
      forwarded_ports(node.vm, host)

      if USE_HOSTMANAGER
        host_aliases = host.key?('aliases') ? host['aliases'].join(' ') : ''
        host_alias   = (host.key?('domain') || global.key?('domain')) ? host['hostname'] : ''
        if !([host_alias,host_aliases].reject(&:empty?).join(' ')).empty?
          node.hostmanager.aliases = [host_alias,host_aliases].reject(&:empty?).join(' ')
        end
        # Vagrant adds the hostname to the loopback address 127.0.0.1, on Ubuntu also to 127.0.1.1
        # https://github.com/hashicorp/vagrant/issues/7263
        node.vm.provision :shell, run: "always", name: "Removing hostname from loopback address...",
                          inline: "sed -i'' '/^127.0.[0-1].1.*#{host['hostname']}$/d' /etc/hosts"
      end

      # Synched folders setup
      synced_folders(node.vm, host, global)
      if run_locally? and (controlhost or i == hosts.length)
        # Mandatory in "ansible_local" mode for multihost environments with Ansible controlhost
        node.vm.synced_folder "."         , "/vagrant"         , type: "virtualbox", mount_options: ['dmode=0775', 'fmode=0664']
        node.vm.synced_folder "./.vagrant", "/vagrant/.vagrant", type: "virtualbox", mount_options: ['dmode=0700', 'fmode=0600']
      end

      # SSH setup for multimaster environments, copy the insecure private SSH key...
      node.vm.provision :file, source: "./.vagrant/insecure_private_key", destination: ".ssh/id_rsa"
      # and make sure it has the right permissions
      node.vm.provision :shell, name: "Provisioning insecure private SSH key...",
                        inline: "chmod 600 /home/#{VAGRANT_USER}/.ssh/id_rsa"

      # File provisioners
      if RUN_FILE_PROVISIONER
        file_provisioners(node.vm, host, global)
      end

      # Shell provisioners
      if RUN_SHELL_PROVISIONER
        # Inline Shell provisioners
        inline_shell_provisioners(node.vm, host, global)
        inline_shell_provisioners_always(node.vm, host, global)
        inline_shell_provisioners_never(node.vm, host, global)
        # Script/URL Shell provisioners
        shell_provisioners(node.vm, host, global)
        shell_provisioners_always(node.vm, host, global)
        shell_provisioners_never(node.vm, host, global)
      end

      @ui.warn "#{node.vm.hostname}: #{node.vm.box} [Memory: #{vm_memory}, CPUs: #{vm_cpus}]"

      # VirtualBox Provider
      node.vm.provider :virtualbox do |vb|
        # This options changes the properties of a registered virtual machine which is not running.
        # Standard Virtualbox VM options
        vb.gui = vm_gui
        vb.auto_nat_dns_proxy = vm_auto_nat_dns_proxy
        vb.customize ["modifyvm", :id, "--name",   vm_name]
        vb.customize ["modifyvm", :id, "--groups", vm_groups]
        vb.customize ["modifyvm", :id, "--memory", vm_memory]
        vb.customize ["modifyvm", :id, "--cpus",   vm_cpus]
        # Advanced Virtualbox VM options
        merge_vm_parameters(host, global, vb)
        # Add additional "Standard" disks to the VM (assumes a SATA controller), beside the system disk assumed to be on SATA port 0
        merge_vm_disks(host, global, vb, sata_controller)
      end # node.vm.provider

      # Only execute the Ansible provisioner once, when all the machines are up and ready
      if i == hosts.length && RUN_ANSIBLE_PROVISIONER
        @ui.warn "Ansible Local Mode: #{run_locally?}, Ansible Playbook: #{ANSIBLE_PLAYBOOK}"
        provision_ansible(node.vm, host, global)
      end

      # TODO: Implement trigger.run_remote
      # node.trigger.after :up do |trigger|
      #   trigger.run_remote = {inline: "ip addr; cat /etc/hosts"}
      # end

      # Present the user with the post_up message
      message = host['postup_message'] if host.key?('postup_message')
      node.vm.post_up_message = "#{node.vm.box} [Memory: #{vm_memory}, CPUs: #{vm_cpus}]\n\n" + message.to_s

    end # config.vm.define
  end # hosts.each
end # Vagrant.configure

# -*- mode: ruby -*-
# vi: ft=ruby :
