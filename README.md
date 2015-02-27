# dev-lxc

A tool for creating Chef server clusters using LXC containers.

Using [ruby-lxc](https://github.com/lxc/ruby-lxc) it builds a standalone server or
tier cluster composed of a backend and multiple frontends with round-robin DNS resolution.

The dev-lxc tool is well suited as a tool for support related work, customized
cluster builds for demo purposes, as well as general experimentation and exploration.

### Features

1. LXC 1.0 Containers - Resource efficient servers with fast start/stop times and standard init
2. Btrfs - Efficient storage backend provides fast, lightweight container cloning
3. Dnsmasq - DHCP networking and DNS resolution
4. Base containers - Containers that are built to resemble a traditional server
5. ruby-lxc - Ruby bindings for liblxc
6. YAML - Simple, customizable definition of clusters; No more setting ENV variables
7. Build process closely models online installation documentation

Its containers, standard init, networking and build process are designed to be similar
to what you would build if you follow the online installation documentation so the end
result is a cluster that is relatively similar to a more traditionally built cluster.

The Btrfs backed clones provide a quick clean slate which is helpful especially for
experimenting and troubleshooting. Or it can be used to build a customized cluster
for demo purposes and be able to bring it up quickly and reliably.

While most of the plumbing is already in place for an HA cluster it actually can't be
used since I haven't been able to get DRBD working inside containers yet.

If you aren't familiar with using containers please read this introduction.

[LXC 1.0 Introduction](https://www.stgraber.org/2013/12/20/lxc-1-0-blog-post-series/)

## Requirements

The `dev-lxc` tool is designed to be used in a platform built by the
[dev-lxc-platform](https://github.com/jeremiahsnapp/dev-lxc-platform) cookbook.

Please follow the dev-lxc-platform usage instructions to create a suitable platform.

The cookbook will automatically install this `dev-lxc` tool.

### Use root

Once you login to the Vagrant VM you should run `sudo -i` to login as the root user.

Consider using `byobu` or `tmux` for a terminal multiplexer as `dev-lxc-platform` README
describes.

### Mounts and Packages (batteries not included)

As described below `dev-lxc` uses a YAML config file for each cluster.

This config file describes what directories get mounted from the Vagrant VM host into
each container. You need to make sure that you configure the mount entries to be
appropriate for your environment.

The same goes for the paths to each Chef package. The paths that are provided in the default
configs are just examples.  You need to make sure that you have each package you want to
use downloaded to appropriate directories that will be available to the container when
it is started.

I recommend downloading the packages to a directory on your workstation.
Then configure the `dev-lxc-platform` `Vagrantfile` to mount that directory in the
Vagrant VM. Finally, configure the cluster's mount entries to mount the Vagrant
VM directory into each container.

## Update `dev-lxc` gem

Run `gem update dev-lxc` inside the Vagrant VM to ensure you have the latest version.

## Background

### Base Containers

One of the key things this tool uses is the concept of "base" containers.

`dev-lxc` creates base containers with a "p-", "s-" or "u-" prefix on the name to distinguish it as
a "platform", "shared" or "unique" base container.

Base containers are then snapshot cloned using the btrfs filesystem to very quickly
provide a lightweight duplicate of the base container. This clone is either used to build
another base container or a container that will actually be run.

During a cluster build process the base containers that get created fall into three categories.

1. Platform

    The platform base container is the first to get created and is identified by the
	"p-" prefix on the container name.

    `DevLXC#create_base_platform` controls the creation of a platform base container.

    This container provides the chosen OS platform and version (e.g. p-ubuntu-1404).
	A typical LXC container has minimal packages installed so `dev-lxc` makes sure that the
	same packages used in Chef's [bento boxes](https://github.com/opscode/bento) are
	installed to provide a more typical server environment.
	A few additional packages are also installed.

    *Once this platform base container is created there is rarely a need to delete it.*

2. Shared

    The shared base container is the second to get created and is identified by the
	"s-" prefix on the container name.

    `DevLXC::ChefServer#create_base_server` controls the creation of a shared base container.

    Chef packages that are common to all servers in a Chef cluster, such as Chef server,
	opscode-reporting and opscode-push-jobs-server are installed using `dpkg` or `rpm`.

    Note the manage package will not be installed at this point since it is not common to all
	servers (i.e. it does not get installed on backend servers).

    The name of this base container is built from the names and versions of the Chef packages that
	get installed which makes this base container easy to be reused by another cluster that is
	configured to use the same Chef packages.

    *Since no configuration actually happens yet there is rarely a need to delete this container.*

3. Unique

    The unique base container is the last to get created and is identified by the
	"u-" prefix on the container name.

    `DevLXC::ChefServer#create` controls the creation of a unique base container.

    Each unique Chef server (e.g. standalone, backend or frontend) is created.

    * The specified hostname is assigned.
	* dnsmasq is configured to reserve the specified IP address for the container's MAC address.
	* A DNS entry is created in dnsmasq if appropriate.
	* All installed Chef packages are configured.
	* Test users and orgs are created.
	* The opscode-manage package is installed and configured if specified.

    After each server is fully configured a snapshot clone of it is made resulting in the server's
	unique base container. These unique base containers make it very easy to quickly recreate
	a Chef cluster from a clean starting point.

#### Destroying Base Containers

When using `dev-lxc cluster destroy` to destroy an entire Chef cluster or `dev-lxc server destroy [NAME]`
to destroy a single Chef server you have the option to also destroy any or all of the three types
of base containers associated with the cluster or server.

Either of the following commands will list the options available.

    dev-lxc cluster help destroy

    dev-lxc server help destroy

Of course, you can also just use the standard LXC commands to destroy any container.

    lxc-destroy -n [NAME]

#### Manually Create a Platform Base Container

Platform base containers can be used for purposes other than building clusters. For example, they can
be used as Chef nodes for testing purposes.

You can see a menu of platform base containers this tool can create by using the following command.

    dev-lxc create

The initial creation of platform base containers can take awhile so let's go ahead and start creating
an Ubuntu 12.04 base container now.

    dev-lxc create p-ubuntu-1404

### Cluster Config Files

dev-lxc uses a YAML configuration file to define a cluster.

The following command generates sample config files for various cluster topologies.

	dev-lxc cluster init

`dev-lxc cluster init tier` generates the following file:

    platform_container: p-ubuntu-1404
    topology: tier
    api_fqdn: chef-tier.lxc
    mounts:
      - /dev-shared dev-shared
    packages:
      server: /dev-shared/chef-packages/cs/chef-server-core_12.0.3-1_amd64.deb
    #  reporting: /dev-shared/chef-packages/reporting/opscode-reporting_1.2.3-1_amd64.deb
    #  push-jobs-server: /dev-shared/chef-packages/push-jobs-server/opscode-push-jobs-server_1.1.6-1_amd64.deb
    #  manage: /dev-shared/chef-packages/manage/opscode-manage_1.9.0-1_amd64.deb
    servers:
      be-tier.lxc:
        role: backend
        ipaddress: 10.0.3.202
        bootstrap: true
      fe1-tier.lxc:
        role: frontend
        ipaddress: 10.0.3.203
    #  fe2-tier.lxc:
    #    role: frontend
    #    ipaddress: 10.0.3.204

This config defines a tier cluster consisting of a single backend and a single frontend.
A second frontend is commented out to conserve resources.

If you uncomment the second frontend then both frontends will be created and dnsmasq will
resolve the `api_fqdn` [chef-tier.lxc](chef-tier.lxc) to both frontends using a round-robin policy.

The config file is very customizable. You can add or remove mounts, packages or servers,
change ip addresses, change server names, change the base_platform and more.

Make sure the mounts and packages represent paths that are available in your environment.

### Managing Multiple Clusters

By default, `dev-lxc` looks for a `dev-lxc.yml` file in the present working directory.
You can also specify a particular config file as an option for most dev-lxc commands.

I use the following strategy to avoid specifying each cluster's config file while managing multiple clusters.

	mkdir -p ~/clusters/{clusterA,clusterB}
	dev-lxc cluster init tier > ~/clusters/clusterA/dev-lxc.yml
	dev-lxc cluster init standalone > ~/clusters/clusterB/dev-lxc.yml
	cd ~/clusters/clusterA && dev-lxc cluster start  # starts clusterA
	cd ~/clusters/clusterB && dev-lxc cluster start  # starts clusterB

### Maintain Uniqueness Across Multiple Clusters

The default cluster configs are already designed to be unique from each other but as you build
more clusters you have to maintain uniqueness across the YAML config files for the following items.

1. Server names and `api_fqdn`

    Server names should really be unique across all clusters.

    Even when cluster A is shutdown, if cluster B uses the same server names when it is created it
	will use the already existing servers from cluster A.

    `api_fqdn` uniqueness only matters when clusters with the same `api_fqdn` are running.

    If cluster B is started with the same `api_fqdn` as an already running cluster A, then cluster B
	will overwrite cluster A's DNS resolution of `api_fqdn`.

    It is easy to provide uniqueness. For example, you can use the following command to replace `-tier`
	with `-1234` in a tier cluster's config.

        sed -i 's/-tier/-1234/' dev-lxc.yml

2. IP Addresses

    IP addresses uniqueness only matters when clusters with the same IP's are running.
	
    If cluster B is started with the same IP's as an already running cluster A, then cluster B
	will overwrite cluster A's DHCP reservation of the IP's but dnsmasq will still refuse to
	assign the IP's to cluster B because they already in use by cluster A. dnsmasq then assigns
	random IP's from the DHCP pool to cluster B leaving it in an unexpected state.

    The `dev-lxc-platform` creates the IP range 10.0.3.150 - 254 for DHCP reserved IP's.
	
    Use unique IP's from that range when configuring clusters.

## Usage

### Shorter Commands are Faster (to type that is :)

The root user's `~/.bashrc` file has aliased `dl` to `dev-lxc` for ease of use but for most
instructions in this README I will use `dev-lxc`.

You only have to type enough of a `dev-lxc` subcommand to make it unique.

The following commands are equivalent:

    dev-lxc cluster init standalone
	dl cl i standalone

    dev-lxc cluster start
	dl cl start

    dev-lxc cluster destroy
	dl cl d

### Create and Manage a Cluster

The following instructions will use a tier cluster for demonstration purposes.
The size of this cluster uses about 3GB ram and takes a long time for the first
build of the servers. Feel free to try the standalone config first.

The following command saves a predefined config to dev-lxc.yml.

	dev-lxc cluster init tier > dev-lxc.yml

Starting the cluster the first time takes awhile since it has a lot to build.

The tool automatically creates snapshot clones at appropriate times so future
creation of the cluster's servers is very quick.

	dev-lxc cluster start

[https://chef-tier.lxc](https://chef-tier.lxc) resolves to the frontend.

A test org and user and knife.rb and keys are automatically created in
the bootstrap backend server in /root/chef-repo/.chef for testing purposes.
The `knife-opc` plugin is installed in the embedded ruby environment of the
Private Chef and Enterprise Chef server to facilitate the creation of the test
org and user.

Show the status of the cluster.

    dev-cluster status

Create a local chef-repo with appropriate knife.rb and pem files.
This makes it easy to use knife.

    dev-lxc cluster chef-repo
	cd chef-repo
	knife ssl fetch
	knife client list

Stop the cluster's servers.

	dev-lxc cluster stop

Clones of the servers as they existed immediately after initial installation and configuration
are available so you can destroy the cluster and "rebuild" it within seconds effectively starting
with a clean slate.

    dev-lxc cluster destroy
	dev-lxc cluster start

The abspath subcommand can be used to prepend each server's rootfs path to a particular file.

For example, to edit each server's private-chef.rb file you can use the following command.

    emacs $(dev-lxc cluster abspath /etc/opscode/private-chef.rb)

After modifying the private-chef.rb you could use the run_command subcommand to tell each server
to run `private-chef-ctl reconfigure`.

    dev-lxc cluster run_command 'private-chef-ctl reconfigure'

Use the following command to destroy the cluster's servers and also destroy their unique and shared
base containers so you can build them from scratch.

    dev-lxc cluster destroy -u -s

You can also run most of these commands against individual servers by using the server subcommand.

    dev-lxc server ...

### Using the dev-lxc library

dev-lxc can also be used as a library if preferred.

    irb(main):001:0> require 'yaml'
	irb(main):002:0> require 'dev-lxc'
	irb(main):003:0> cluster = DevLXC::ChefCluster.new(YAML.load(IO.read('dev-lxc.yml')))
	irb(main):004:0> cluster.start
	irb(main):005:0> server = DevLXC::ChefServer.new("fe1-tier.lxc", YAML.load(IO.read('dev-lxc.yml')))
	irb(main):006:0> server.stop
	irb(main):007:0> server.start
	irb(main):008:0> server.run_command("private-chef-ctl reconfigure")
	irb(main):009:0> cluster.destroy

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
