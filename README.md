To install docker and make the docker image of discourse follow these steps:

1. Older versions of Docker were called docker or docker-engine. If these are installed, uninstall them: 

	sudo apt-get remove docker docker-engine
	
2. Docker needs to use the aufs storage drivers:

	sudo apt-get update
	
	sudo apt-get install \
    	  linux-image-extra-$(uname -r) \
    	  linux-image-extra-virtual
	  
3. Install the latest stable Docker:

	sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 \
     		--recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
		
	sudo sh -c "echo deb https://get.docker.com/ubuntu docker main \
           	> /etc/apt/sources.list.d/docker.list"
		
	sudo apt-get update
	
	sudo apt-get install docker docker-engine 
	
4. Clone the git repository hosting the Discourse Docker configuration into the /var/discourse directory:

	sudo -s
	
	mkdir /var/discourse
	
	git clone https://github.com/discourse/discourse_docker.git /var/discourse
	
	cd /var/discourse
	
5. Configure Discourse

	./discourse-setup
	
	Answer the following questions when prompted:

	Hostname for your Discourse? [discourse.example.com]:
	
	Email address for admin account? [me@example.com]:
	
	SMTP server address? [smtp.example.com]: 
	
	SMTP user name? [postmaster@discourse.example.com]: 
	
	SMTP port [587]:
	
	SMTP password? []: 

	Edit containers/app.yml
	by editing the answers for the things above in env section
	
6. For the changes to be in effect:

	./launcher rebuild app
	
7. If you need to use discourse remotely for your domain then install and configure Nginx on the server:

	sudo apt-get install -y software-properties-common

	sudo add-apt-repository -y ppa:nginx/stable
	
	sudo apt-get update
	
	sudo apt-get install -y nginx
	
Create your Nginx configuration. Created /etc/nginx/sites-available/??????????.conf:

upstream discourse

{

	server 127.0.0.1:80;
	
}

server 

{

   listen 80 default_server;
   
   root /var/www/??????.????????.com/public;
   
   index index.html index.htm;

    access_log /var/log/nginx/??????.????????.com.log;
    error_log  /var/log/nginx/??????.????????.com-error.log error;

    server_name ??????.????????.com;

    charset utf-8;

    location / {
        include proxy_params;
        proxy_pass http://discourse;

        proxy_redirect off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}



9. For nginx setup:

	# Remove default config
	sudo rm /etc/nginx/sites-enabled/default

	# Symlink our new config
	sudo ln -s /etc/nginx/sites-available/??????.conf \
           /etc/nginx/sites-enabled/??????.conf

	# Test configuration
	sudo service nginx configtest

	# If configtest checks out:
	sudo service nginx reload

10. Now run the discourse on the browser and register for discourse


### About

- [Docker](https://docker.com/) is an open source project to pack, ship and run any Linux application in a lighter weight, faster container than a traditional virtual machine.

- Docker makes it much easier to deploy [a Discourse forum](https://github.com/discourse/discourse) on your servers and keep it updated. For background, see [Sam's blog post](http://samsaffron.com/archive/2013/11/07/discourse-in-a-docker-container).

- The templates and base image configure Discourse with the Discourse team's recommended optimal defaults.

### Getting Started

The simplest way to get started is via the **standalone** template, which can be installed in 30 minutes or less. For detailed install instructions, see

https://github.com/discourse/discourse/blob/master/docs/INSTALL-cloud.md

### Directory Structure

#### `/cids`

Contains container ids for currently running Docker containers. cids are Docker's "equivalent" of pids. Each container will have a unique git like hash.

#### `/containers`

This directory is for container definitions for your various Discourse containers. You are in charge of this directory, it ships empty.

#### `/samples`

Sample container definitions you may use to bootstrap your environment. You can copy templates from here into the containers directory.

#### `/shared`

Placeholder spot for shared volumes with various Discourse containers. You may elect to store certain persistent information outside of a container, in our case we keep various logfiles and upload directory outside. This allows you to rebuild containers easily without losing important information. Keeping uploads outside of the container allows you to share them between multiple web instances.

#### `/templates`

[pups](https://github.com/samsaffron/pups)-managed templates you may use to bootstrap your environment.

#### `/image`

Dockerfiles for Discourse; see [the README](image/README.md) for further details.

The Docker repository will always contain the latest built version at: https://hub.docker.com/r/discourse/discourse/, you should not need to build the base image.

### Launcher

The base directory contains a single bash script which is used to manage containers. You can use it to "bootstrap" a new container, enter, start, stop and destroy a container.

```
Usage: launcher COMMAND CONFIG [--skip-prereqs]
Commands:
    start:      Start/initialize a container
    stop:       Stop a running container
    restart:    Restart a container
    destroy:    Stop and remove a container
    enter:      Use docker exec to enter a container
    logs:       Docker logs for container
	memconfig:  Configure sane defaults for available RAM
    bootstrap:  Bootstrap a container for the config based on a template
    rebuild:    Rebuild a container (destroy old, bootstrap, start new)
```

If the environment variable "SUPERVISED" is set to true, the container won't be detached, allowing a process monitoring tool to manage the restart behaviour of the container.

### Container Configuration

The beginning of the container definition can contain the following "special" sections:

#### templates:

```
templates:
  - "templates/cron.template.yml"
  - "templates/postgres.template.yml"
```

This template is "composed" out of all these child templates, this allows for a very flexible configuration structure. Furthermore you may add specific hooks that extend the templates you reference.

#### expose:

```
expose:
  - "2222:22"
  - "127.0.0.1:20080:80"
```

Expose port 22 inside the container on port 2222 on ALL local host interfaces. In order to bind to only one interface, you may specify the host's IP address as `([<host_interface>:[host_port]])|(<host_port>):<container_port>[/udp]` as defined in the [docker port binding documentation](http://docs.docker.com/userguide/dockerlinks/)


#### volumes:

```
volumes:
  - volume:
      host: /var/discourse/shared
      guest: /shared

```

Expose a directory inside the host to the container.

#### links:
```
links:
  - link:
      name: postgres
      alias: postgres
```

Links another container to the current container. This will add `--link postgres:postgres`
to the options when running the container.

### Upgrading Discourse

The Docker setup gives you multiple upgrade options:

1. Use the front end at http://yoursite.com/admin/upgrade to upgrade an already running image.

2. Create a new base image manually by running:
  - `./launcher rebuild my_image`

### Single Container vs. Multiple Container

The samples directory contains a standalone template. This template bundles all of the software required to run Discourse into a single container. The advantage is that it is easy.

The multiple container configuration setup is far more flexible and robust, however it is also more complicated to set up. A multiple container setup allows you to:

- Minimize downtime when upgrading to new versions of Discourse. You can bootstrap new web processes while your site is running and only after it is built, switch the new image in.
- Scale your forum to multiple servers.
- Add servers for redundancy.
- Have some required services (e.g. the database) run on beefier hardware.

If you want a multiple container setup, see the `data.yml` and `web_only.yml` templates in the samples directory. To ease this process, `launcher` will inject an env var called `DISCOURSE_HOST_IP` which will be available inside the image.

WARNING: In a multiple container configuration, *make sure* you setup iptables or some other firewall to protect various ports (for postgres/redis).
On Ubuntu, install the `ufw` or `iptables-persistent` package to manage firewall rules.

### Email

For a Discourse instance to function properly Email must be set up. Use the `SMTP_URL` env var to set your SMTP address, see sample templates for an example. The Docker image does not contain postfix, exim or another MTA, it was omitted because it is very tricky to set up correctly.

### Troubleshooting

View the container logs: `./launcher logs my_container`

Spawn a shell inside your container using `./launcher enter my_container`. This is the most foolproof method if you have host root access.

If you see network errors trying to retrieve code from `github.com` or `rubygems.org` try again - sometimes there are temporary interruptions and a retry is all it takes.

Behind a proxy network with no direct access to the Internet? Add proxy information to the container environment by adding to the existing `env` block in the `container.yml` file:

```yaml
env:
    …existing entries…
    HTTP_PROXY: http://proxyserver:port/
    http_proxy: http://proxyserver:port/
    HTTPS_PROXY: http://proxyserver:port/
    https_proxy: http://proxyserver:port/
```

### Security

Directory permissions in Linux are UID/GID based, if your numeric IDs on the
host do not match the IDs in the guest, permissions will mismatch. On clean
installs you can ensure they are in sync by looking at `/etc/passwd` and
`/etc/group`, the Discourse account will have UID 1000.


### Advanced topics

- [Setting up SSL with Discourse Docker](https://meta.discourse.org/t/allowing-ssl-for-your-discourse-docker-setup/13847)
- [Multisite configuration with Docker](https://meta.discourse.org/t/multisite-configuration-with-docker/14084)
- [Linking containers for a multiple container setup](https://meta.discourse.org/t/linking-containers-for-a-multiple-container-setup/20867)
- [Using Rubygems mirror to improve connection problem in China](https://meta.discourse.org/t/replace-rubygems-org-with-taobao-mirror-to-resolve-network-error-in-china/21988/1)

### Developing with Vagrant

If you are looking to make modifications to this repository, you can easily test
out your changes before committing, using the magic of
[Vagrant](http://vagrantup.com).  Install Vagrant as per [the default
instructions](http://docs.vagrantup.com/v2/installation/index.html), and
then run:

    vagrant up

This will spawn a new Ubuntu VM, install Docker, and then await your
instructions.  You can then SSH into the VM with `vagrant ssh`, become
`root` with `sudo -i`, and then you're right to go.  Your live git repo is
already available at `/var/discourse`, so you can just `cd /var/discourse`
and then start running `launcher`.


License
===
MIT
