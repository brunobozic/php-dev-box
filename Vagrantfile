required_plugins = %w( vagrant-cachier )
required_plugins.each do |plugin|
    exec "vagrant plugin install #{plugin};vagrant #{ARGV.join(" ")}" unless Vagrant.has_plugin? plugin || ARGV[0] == 'plugin'
end
require 'yaml'

vagrant_root = File.expand_path("..", __FILE__)
vagrant_method = ARGV[0]
vagrantversion = Vagrant::VERSION
output = Vagrant::UI::Colored.new
color_info = {:color => :cyan}

output.info("vagrant_root is #{vagrant_root}", color_info)
output.info("vagrant_method is #{vagrant_method}", color_info)
output.info("vagrantversion is #{vagrantversion}", color_info)

# Find current box name
for machine in Dir["#{vagrant_root}/.vagrant/machines/*/"]
    if File.exist?("#{machine}/virtualbox/index_uuid")
        current_box_name = File.basename(machine)
        current_box_id = File.read("#{machine}/virtualbox/index_uuid")
        break
    end
end

output.info("current_box_name is #{current_box_name}", color_info)
output.info("vagrant_root is #{vagrant_root}", color_info)

# Load default config YAML
system_config_file = "#{vagrant_root}/system.yml"
system_options = YAML.load_file(system_config_file)
default_config_file = "#{vagrant_root}/config.default.yml"
options = YAML.load_file(default_config_file)
config_file = "#{vagrant_root}/config.yml"

# Load custom config file if it exists
if File.exist?(config_file) == true
    custom_options = YAML.load_file(config_file)
end

# Merge defualts with custom options if they are defined
if custom_options.is_a?(Hash)
    options = options.merge(custom_options)
end

# Merge system settings into options
options = options.merge(system_options)

# If you are running something else than up or provision, default box_name to current name
current_box_name = 'php-dev-box'
box_name = current_box_name

output.info("vagrant_root is #{box_name}", color_info)

#If your Vagrant version is lower than 1.5, you can still use this provisioning
#by commenting or removing the line below and providing the config.vm.box_url parameter,
#if it's not already defined in this Vagrantfile. Keep in mind that you won't be able
#to use the Vagrant Cloud and other newer Vagrant features.
Vagrant.require_version ">= 1.5"

# Check to determine whether we're on a windows or linux/os-x host,
# later on we use this to launch ansible in the supported way
# source: https://stackoverflow.com/questions/2108727/which-in-ruby-checking-if-program-exists-in-path-from-ruby
def which(cmd)
    exts = ENV['PATHEXT'] ? ENV['PATHEXT'].split(';') : ['']
    ENV['PATH'].split(File::PATH_SEPARATOR).each do |path|
        exts.each { |ext|
            exe = File.join(path, "#{cmd}#{ext}")
            return exe if File.executable? exe
        }
    end
    return nil
end

## Create the repo path if it doesn't exist
#if options['repo_path'][0, 1] == '/'
#    repo_path = File.expand_path(options['repo_path'])
#elsif
#    repo_path = File.expand_path(vagrant_root + '/' + options['repo_path'])
#end

repo_path = File.expand_path(options['repo_path'])
per_portal_staging_path_on_host = File.expand_path(options['per_portal_staging_path_host'])

output.info("repo_path before expand is #{repo_path}", color_info)
output.info("per_portal_staging_path_on_host before expand is #{per_portal_staging_path_on_host}", color_info)

if File.directory?(repo_path) == false
    `mkdir -p #{repo_path}`
end

if File.directory?(per_portal_staging_path_on_host) == false
    `mkdir -p #{per_portal_staging_path_on_host}`
end

output.info("repo_path after expand is #{repo_path}", color_info)
output.info("per_portal_staging_path_on_host after expand is #{per_portal_staging_path_on_host}", color_info)

Vagrant.configure("2") do |config|

    config.vm.provider :virtualbox do |v|
        v.name = "default"
        v.customize [
            "modifyvm", :id,
            "--name", "default",
            "--memory", 2048,
            "--natdnshostresolver1", "on",
            "--cpus", 1,
            "--ostype", "Ubuntu_64",
            "--natdnsproxy1", "on",
            "--natdnshostresolver1", options['nat_dns_host_resolver'] ? "on" : "off"
        ]
    end
    
    # Most examples don't, so I don't
    config.ssh.insert_key = false

    # Prevent TTY Errors
    config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"

    # Deny SSH Agent Forward from The Box
    config.ssh.forward_agent = false

    config.vm.hostname = "php-dev-box"

    config.vm.box = "ubuntu/trusty64"
    
    config.vm.box_url = "https://vagrantcloud.com/ubuntu/boxes/trusty64/versions/14.04/providers/virtualbox.box"
    
    config.vm.network :private_network, ip: "192.168.31.91"
    config.ssh.forward_agent = true
    config.vm.provision :shell do |shell|
        shell.inline = "chmod -R 777 /var/log/"
    end
 
    if Vagrant.has_plugin?("vagrant-cachier")
        # Configure cached packages to be shared between instances of the same base box.
        # More info on http://fgrehm.viewdocs.io/vagrant-cachier/usage
        config.cache.scope = :box
    end

    # If ansible is in your path it will provision from your HOST machine
    # If ansible is not found in the path it will be instaled in the VM and provisioned from there
    if which('ansible-playbook')
        config.vm.provision "ansible" do |ansible|
            ansible.playbook = "ansible/playbook.yml"
            ansible.inventory_path = "ansible/inventories/dev"
            ansible.limit = 'all'
            ansible.verbose = options['ansible_verbosity']
            ansible.sudo = true
        end
    #else
    end

    output.info("nfs_mount_options is #{options['nfs_mount_options']}", color_info)
    output.info("repo_path is #{options['repo_path']}", color_info)
 
    output.info("per_portal_staging_path_host is #{options['per_portal_staging_path_host']}", color_info)
    output.info("per_portal_staging_path_vagrant is #{options['per_portal_staging_path_vagrant']}", color_info)

    config.vm.provision :shell, path: "ansible/windows.sh", args: ["default"]
    config.vm.provision :shell, :inline => "echo \"UTC\" | sudo tee /etc/timezone && dpkg-reconfigure --frontend noninteractive tzdata"

    config.vm.synced_folder repo_path, '/home/vagrant/my-app-repository', mount_options: [options['nfs_mount_options_safe']]
    config.vm.synced_folder options['per_portal_staging_path_host'], options['per_portal_staging_path_vagrant'], mount_options: [options['nfs_mount_options_safe']]
end