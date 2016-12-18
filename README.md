# DNS and Docker

Tired of editing `/etc/hosts` when testing docker containers from the host? 

For docker containers to resolve each other using (service and container) names, an embedded Docker DNS can be utilized. 

You can set up `dnsmasq` or `dnsdock` and use those to resolve names from your host machine where the docker daemon runs.

## HOW

This repository contains two directories demonstrating two approaches for how to do this (using dnsmasq and dnsdock respectively).For name resolution, the classic solution is `dnsmasq` but there is also a more Docker-tailored variant that can be used; `dnsdock`, which seems to be popular as an alternative to "skydns" and other heavier service discovery solutions.

This setup works also using the convenient `docker-compose` wrapper, if the .yml file starts with "version: '2'". In this project, you will see that each directory has a `docker-compose.yml` file defining docker micro-services and a `Makefile` that manages the services (start, stop) and runs tests etc.

For making sure that the host running the docker daemon also can resolve a name in the same way as names are resolved internally between containers, the host machine running the containers need to be setup to resolve names not only using its default configured DNS resolution, but also in addition, using a separate Docker DNS container running locally on the host. 

In practice, on a Linux host, these are possible configs to achieve what was described above:

- The docker daemon needs to be configured to run on the `docker0` bridge interface (with a static ip: 172.17.0.1) and then `dnsmasq` would bind to port 53 udp of this ip. For details, see the next section!
- The host machine needs to be configured to use `resolvconf` and to add the Docker internal DNS (/etc/resolvconf/resolv.conf.d/head needs to have `nameserver 172.17.0.1`). You may need to do "sudo dpkg-reconfigure resolvconf".
- Possibly, you may want to edit "/etc/dhcp/dhclient.conf" and add "prepend domain-name-servers 172.17.0.1;". But possibly not, seems to work without it.

## Configuring the docker daemon on the host

Instructions below are taken from Docker Hub pages for the "dnsdock" image: https://hub.docker.com/r/tonistiigi/dnsdock/:

- If you use systemd (present on Fedora and recent Ubuntu versions including latest Mint editions), edit /lib/systemd/system/docker.service and add the options to the command you will see in the ExecStart section, then run sudo systemctl daemon-reload. Or even better, follow the official instructions from https://docs.docker.com/engine/admin/systemd/ and add your own .conf file in the recommended location, then system updates will not be able to overwrite your config settings.
- If you do not, Open file /etc/default/docker and add --bip=172.17.0.1/24 --dns=172.17.0.1 to DOCKER_OPTS variable.

Restart docker daemon after you have done that (sudo service docker restart or sudo systemctl restart docker.service respectively depending on your linux distro).

## Walkthrough, step-by-step

To figure out which nameservers you use currently, you can use some tool suitable for your platform. On Linux it could for example be a command like: `nm-tool | grep DNS` or for other ways to do it, see http://askubuntu.com/questions/152593/command-line-to-list-dns-servers-used-by-my-system.

For your host machine to be able to use the Docker DNS server a few changes has to be made for the docker daemon and the IP of the Docker DNS server needs to be added to the nameserver list used by your host - on Linux this is done through the resolvconf configurations. To configure the Docker Daemon, create a file `/etc/docker/daemon.json` with the following content:

	{
		"dns": ["172.17.0.1", "your.corporate.network.nameserver1", "your.corporate.network.nameserver2", "8.8.8.8", 8.8.4.4"],
		"bip": "172.17.0.1/24"
	}

The first IP is the IP of the dnsdock container. The second and third are nameserver used in your corporate network. The last two are public DNS server IPs offered by Google. To add the DNSdock's IP in the first line of resolv.conf in your host
machine, do NOT edit that file, instead ensure you have 'resolvconf' package installed and add the following line to `/etc/resolvconf/resolv.conf.d/head`:

	nameserver 172.17.0.1

Then restart the service resolvconf

	sudo service resolvconf restart # or equiv depending on your distro

The DNSdock container IP is then added to resolv.conf, you can verify it with:

	cat /etc/resolv.conf

This should enable other services to identify each other using domain name besides container name and also be able to reach the Internet from within the containers.

# Should you be using dnsmasq and/or dnsdock?

## dnsmasq

Using this approach, the key is to set "container_name" in docker-compose.yml!

Look at the `docker-compose.yml` file for details. 

Use the Makefile to test that you have DNS working:

	cd dnsmasq
	make up
	make test-dns-internal
	make test-dns-external
	make test-dns-debug
	make clean

Further reading:

- http://stackoverflow.com/questions/23012273/setting-up-docker-dnsmasq
- https://hub.docker.com/r/andyshinn/dnsmasq/

## dnsdock

Unlike with `dnsmasq`, with `dnsdock`, the container_name is not everything and you have more at your disposal...

You also get service discovery using a REST API:

	curl -s http://dnsdock.docker/services | json_pp 

And you get aliases and other stuff too, ie the same container can be known under several different names.

### Default name composition using dnsdock

The format for a request matching a container is:

	<anything>.<container-name>.<image-name>.<environment>.<domain>.

Details for components in that formula:

- environment (defaults to "") and domain (defaults to "docker") are static suffixes that are set on startup.
- image-name is last part of the image tag used when starting the container.
- container-name alphanumerical part of container name.

You can always leave out parts from the left side. If multiple containers match then they are all returned. Wildcard requests are also supported, i.e "dig *.docker +short" will list all ips matching that DNS query.

### Warning - use the right tag

NB: The "tonistiigi/dnsdock:latest" docker image is no good. 

You need something like this in the `docker-compose.yml` file:

	  dnsdock:
	    image: tonistiigi/dnsdock:amd64-1.13.1
	    ports:
	      - 172.17.0.1:53:53/udp
	    volumes:
	      - /var/run/docker.sock:/var/run/docker.sock

### Usage

Do this to test that you have it working:

	cd dnsdock
	make up
	make test-dns-internal
	make test-dns-external
	make test-dns-debug
	make clean

# References

Various links with reading materials

### General

- http://askubuntu.com/questions/130452/how-do-i-add-a-dns-server-via-resolv-conf#134106

### Alternative docker images

- https://hub.docker.com/r/phensley/docker-dns/
- https://hub.docker.com/r/phensley/docker-dns-rest/
- https://github.com/phensley/docker-dns-rest

### FQDN and smooshing hostname

- https://github.com/docker/docker/issues/13378
- https://github.com/docker/docker/pull/20200
- http://superuser.com/questions/962658/how-to-define-domainname-in-a-docker-run-command/1065412#1065412
