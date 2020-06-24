# Raspberry Pi Containerized Selfhosting Cluster

This repository contains the ansible playbooks I used to self host my personal websites,
currently miniflux (a minimalist rss reader) and gitea (a git source forge) on my raspberry pi cluster.

Additional Features:
- Everything is dockerized! This isolates all the services from one another and makes future updates easy.
- Automated backups using duplicty
- Cluster wide log and telemetry monitoring using telegraf, influxdb, and chronograf.
- Automatic letsencrypt ssl encryption using a caddy reverse proxy.

I'll be adding to and improving this over time as I add more services to my cluster.

I built this specifically for my use case with three raspberry pis running specific websites.
Since you're probably not interested in running the same websites as me, this is meant mostly as a starting
point for how to self-host containerized websites on a raspberry pi with docker.

## Installation

This ansible playbook is specifically tailored to run with three raspberry pis imaged with the
raspbain buster minimal edition. Since I made some raspbian specific tweaks, running this script
on anything other that raspbain buster will cause some minor errors.

Before you can run the ansible playbook, static IPv4 ethernet addresses should be assigned to each of the pis,
ideally using your router with dhcp.

I also highly recommend assigning easily readable dns records for those ip addresses, so that if
you want to change ip addresses later all you have to do is update the dns record, rather than
re-run the ansible playbook to re-configure the pis.

You should also change the default password on the raspberry pis and setup ssh keys so ansible can
access the pis.

1. Update the urls in the `example_hosts` file to point to your three raspberry pis
2. Set the passwords and variables in the `example_vars.yml` file. Caddy will automatically use
   letsencrypt to create ssl certificates, so make sure that your gitea and miniflux dns entries are
   good before running the playbook!
3. Run `ansible-playbook -i example_hosts ansible-setup.yml` to configure the raspberry pis.
   Ansible script are mostly immutable, so they can be safely re-run if any errors come up in the progress.
4. Setup the first miniflux user by sshing into the frontend pi and running `sudo docker exec -it miniflux /usr/bin/miniflux -create-admin`
5. Setup the first gittea user by visiting the `https://<gitea url>/install` page.
6. To make sure everything is working, check the metrics using chronograf at `<backend url>:8888`.

## Architecture

I'm using three raspberry pi 3b+ computers for my cluster running raspbian.

- frontend pi: The frontend pi hosts the frontend layers of my cluster, specifically miniflux and gitea, as well and the
  caddy reverse proxy used to access them.
- backend pi: The backend pi is used to host the data storage layers of my cluster. It has a postgres db used by
  miniflux and gitea, as well as influxdb and chronograf for monitoring.
- builder pi: The builder pi is used for testing services and building the docker containers used in the cluster.
  It also also functions as a backup destination for the other two raspberry pis.

Since every processes runs within docker, I mount in directories from the host to persist data across reboots.
Everything in the `/usr/local/cfg` directory is read only configuration mounted into docker containers.
Everything in the `/usr/local/persist` directory is persistent data mounted from docker containers.

## Networking

The three raspberry pis need unrestricted communication with each other.
Right now the postgres db connection and the chronograf web ui are unencrypted,
so it's recommended that the pis are on their own private subnet.

The frontend raspberry pi hosts the frontend webpages, so it needs to have port 80 and 443
exposed to the public internet. For IPv4 this means enabling port forwarding for those
ports on your router. For IPv6 you will probably need to open up incoming traffic on those ports
to the frontend pi on your router's firewall.

Because raspbian defaults to using temporary IPv6 address, the ansible playbook switches dhcpd over
to using permanent hardware based IPv6 addresses.

Wifi and bluetooth are also disabled by the ansible playbook to reduce power consumption

## SSL

Caddy has support for automatically configuring letsencrypt ssl certificates on startup.
I have caddy act as a reverse proxy, where it listens for encrypted traffic,
then forwards the unencrypted traffic on a private docker network to the actual website.

Let's Encrypt uses http requests to verify ownership of a domain. This means that if anything networking is
misconfigured in the cluster, caddy can get stuck in a loop requesting certificates from letsencrypt and get rate
limited for hours/days. I recommend testing with letsencrypt staging first to make sure everything is
configured properly. The staging server doesn't provide browser truster certificates, but it has a higher
rate limit, so you can test the certificate request process is working properly.
To use the staging server, uncomment the staging line in the caddy configuration template in the files directory.

## Backups

The frontend and backend pis are setup to backup the `/usr/local/persist` directory over rsync/ssh with duplicity.
The ansible script creates a `bkup` user on the builder pi, then generates ssh keys on the frontend
and backend, then adds the frontend and backend keys as authorized keys for the backup user.
Duplicity will make incremental backups every other day and create a new full backup once a week.

I opted not to go with encrypted backups because the `/usr/local/persist` directory is unencrypted
on the frontend and the backend. This means that encryption would only provide benefits if somebody
hacked into the builder pi but not the other two, which I consider to be a low risk.


## Monitoring

Each pi runs a telegraf container for telemetry acquisition.
Telegraf sends it's telemetry to an influxdb database on the backend pi.
To visualize the telemetry, a read-only chronograf instance on the backend pi connects to influxdb
to display cluster wide metrics and logs.

To send the docker container and host logs to influxdb, I configured docker to log to journald, then rsyslog to forward the host's journald logs to telegraf.

## Software Updates

The ansible playbook is setup to automatically install updates nightly on all of the pis.
It is not setup automatically reboot after kernel upgrades, so I reccommend checking every week
to see if the pis need to be rebooted.

Any of the containers can be upgraded by changing the image version in the ansible playbook and re-running it.

## Why not Kubernetes?

Despite the fact that I'm hosting a cluster of docker containers, I'm not using kubernetes.
Kubernetes is awesome, but it has a ton a complexity, overhead, and moving parts.

When you're running outside of a cloud environment, you also lose a lot of kubernetes's benefits,
like being able to transfer containers and their associated data between nodes, or automatically
creating a load balancer that routes to your container.

To me, in this situation, the simplicity and reliability I get from running raw docker outweighs the
management benefits of kubernetes.

## Todo

- Setup automated encrypted offsite backups.
- Setup minio/caddy for serving static files.
- Setup a coredns server to act as a dns-over-tls proxy for the network.
- tls encrypted postgres connections
