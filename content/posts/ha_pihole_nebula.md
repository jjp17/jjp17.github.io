---
title: "How to setup high-availability PiHole with Keepalived and Nebula-Sync"
date: 2025-8-14T23:30:00-05:00
# weight: 1
# aliases: ["/first"]
tags: ["pihole", "keepalived"]
categories: ["Selfhosting"]
#author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Two PiHole instances synced with Nebula-Sync"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
#editPost:
#    URL: "https://github.com/jjp17.github.io/content"
#    Text: "Suggest Changes" # edit text
#    appendFilePath: true # to append file path to Edit link
---

## Abstract
We will setup two PiHole instances in an active/passive failover setup using Keepalived. The two PiHole instances will be synced with each other using nebula-sync, where one instance will be the primary and one will be the secondary.

This setup will allow your primary PiHole instance/hardware to fail for whatever reason and automatically switch DNS duty over to the secondary instance, without reconfiguring clients.

The two instances will share a "Virtual IP Address", and PiHole will be served from this address.

This guide will primarily focus on the active/passive failover portion.

## Prerequisites

* Linux knowledge/comfort using the command line
* Two instances on PiHole 5.x setup and running (Ideally on two separate pieces of hardware)
* An `apt` based distro (Install commands will be using `apt`. You can adapt the install commands for your distro)
* A normal switched network (a note about the cloud later in the guide)

## Setup PiHole

The install for PiHole is straightforward so we won't outline the process. You can use the official install guides [here.](https://docs.pi-hole.net/main/basic-install/)
You will have to set this up on each PiHole instance.

## Setup Keepalived

Keepalived is a piece of software that can provide active/passive failover capabilities. All the instances in a keepalived cluster will pass around a `virtual IP`. Whoever is the master will own the virtual IP. 

Clusters are formed by using the `virtual_router_id` parameter in the config. Instances with the same virtual router ID (and correct password) in their config will be part of the same cluster. 

Each instance will continuously check on each other's health. Should one instance in the keepalived cluster fail, the remaining instances in the group will figure out who should be the new leader based on their `priority` and pass them the virtual IP if necessary. 

### On the primary PiHole instance

Install keepalived using apt

```bash
sudo apt install keepalived
```

We'll need to create the config file that keepalived uses for its config. That's typically located in `/etc/keepalived/keepalived.conf`.
```bash
sudo touch /etc/keepalived/keepalived.conf
sudo nano /etc/keepalived/keepalived.conf
```

In the file, paste the following config
```bash
vrrp_instance ha_1 {
        state BACKUP
        interface eth0
        virtual_router_id 69
        priority 255
        advert_int 1
        notify /etc/keepalived/notify_pihole.sh
        authentication {
                auth_type AH
                auth_pass YourPasswordHere
        }
        virtual_ipaddress {
                192.168.1.10
        }   
}
```

You will also need to create a script that fires when keepalived switches an instance's state (i.e. from Master -> Backup or Backup -> Master). We'll put this in a script next to the config file named `notify_pihole.sh`.
```bash
sudo touch /etc/keepalived/notify_pihole.sh
sudo nano /etc/keepalived/notify_pihole.sh
```

In the file, paste in the following script
```bash
#!/bin/bash

TYPE=$1
NAME=$2
STATE=$3

case $STATE in
        "MASTER") pihole restartdns
                  exit 0
                  ;;
        "BACKUP") pihole restartdns
                  exit 0
                  ;;
        "FAULT")  pihole restartdns
                  exit 0
                  ;;
        *)        echo "Unknown State"
                  exit 1
                  ;;
esac
```

With the configuration and script in place, enable and start the keepalived service
```bash
sudo systemctl enable --now keepalived.service
```

Check the status to make sure it's running
```bash
sudo systemctl status keepalived.service
```

If there are any issues, investigate the logs using `journalctl`
```bash
sudo journalctl -fu keepalived.service
```

Now lets quickly run through the config file and notify script so you know what's going on under the hood.

#### keepalived.conf

* `state` defines the state keepalived will start in before talking to other instances.
* `vrrp_instance ha_1` defines an instance of the VRRP protocol. All of our configuration goes inside this instance
* `interface` tells keepalived what network interface it should run on. You should change this if your instances don't use eth0. Make sure this is the interface PiHole uses too
* `virtual_router_id` is the ID value instances in the cluster must share to be considered "in the same cluster". The primary and secondary PiHole instances should have the same value.
* `priority` tells keepalived and the rest of the cluster how important this particular instance is. The higher the priority (up to 255), the more important the instance is. Of all the running instances that are communicating with each other, the one with the highest priority holds and serves the virtual IP address. 
* `advert_int` specifies the frequency that advertisements are sent in seconds.
* `notify` specifies the path to a script to run when keepalived changes state. When it runs the script, it also passes along three arguments: Type, Name, and End State of the transition.
* `authentication` block sets up authentication between instances. In this case we set it to "AH" (Authentication Headers), and provide a password (which you should change). It supports a max of 8 characters before it starts to truncate the password
* `virtual_ip_address` block defines the Virtual IP Addresses that will be active on the master keepalived machine

#### notify_pihole.sh

The notify_pihole.sh script is called by keepalived whenever it switches its state. It also provides those three arguments mentioned above.

The most important argument is the end state argument. It tells the script what state keepalived will end up in as it switches. You can get fancy and have different cases for Master, Backup and Error states, but in this setup simply restarting PiHole's DNS is sufficient for all cases.

### On the secondary PiHole instance

Setup on the secondary instance is identical to the primary with the exception of the priority. The secondary instance should be set at a lower priority than the primary. This ensures that when both instances are healthy, the primary is the one serving PiHole.

Create the conf file
```bash
sudo touch /etc/keepalived/keepalived.conf
sudo nano /etc/keepalived/keepalived.conf
```

Paste in the config **(notice the lower priority value here)**
```bash
vrrp_instance ha_1 {
        state BACKUP
        interface eth0
        virtual_router_id 69
        priority 254
        advert_int 1
        notify /etc/keepalived/notify_pihole.sh
        authentication {
                auth_type AH
                auth_pass YourPasswordHere
        }
        virtual_ipaddress {
                192.168.1.10
        }   
}
```

Create the notify script
```bash
sudo touch /etc/keepalived/notify_pihole.sh
sudo nano /etc/keepalived/notify_pihole.sh
```

Paste in the script
```bash
#!/bin/bash

TYPE=$1
NAME=$2
STATE=$3

case $STATE in
        "MASTER") pihole restartdns
                  exit 0
                  ;;
        "BACKUP") pihole restartdns
                  exit 0
                  ;;
        "FAULT")  pihole restartdns
                  exit 0
                  ;;
        *)        echo "Unknown State"
                  exit 1
                  ;;
esac
```

### A note about VRRP and the cloud

This guide assumes you have two instances of PiHole that are connected to a normal, switched network (whether that be a virtual switch or physical). The protocol behind keepalived is VRRP and VRRP needs a multicast-enabled network to work properly. Most public cloud providers do not support multicast and you may encounter some difficulty should you try this there. 

You can however, enable unicast for keepalived if you are running in the cloud. You'll need to add the following to both of your keepalived configs:
```bash
        unicast_src_ip 192.168.1.11
        unicast_peer {
                192.168.1.12
        }
```

* `unicast_src_ip` should be the IP address of the instance you are setting the config for.\
* `unicast_peer` should be the IP address of the other instance

You will not be able to simply use the virtual IP address specified in your config however. You'll have to get fancy with lifting and shifting IPs/NICs in your cloud providers environment using the notify script.

In the case of AWS for example, you'll need to create and attach a secondary "floating" NIC to the master instance. If the master instance fails, your notify script on the secondary instance should be able to makes calls to AWS CLI to:

1. Locate what instance the "floating" NIC is attached to
2. Detach the "floating" NIC from the the instance it's currently attached to
3. Attach the "floating" NIC to itself
4. Bring up the interface/IP address associated with the "floating" NIC
5. Restart PiHole

Your notify script should also be able to do this on the master instance when it eventually comes back online.

## Setup Nebula-Sync

Now that keepalived is setup, we can install nebula-sync

nebula-sync is a utility that allows two PiHole instances to keep ad-lists, domain whitelists, clients/groups, Local DNS/CNAME records, and DHCP assignments in sync with each other

At a (rough) high level, you will:
* Create an app password for each PiHole instance
* Tweak security settings on the secondary PiHole instance
* Download the Nebula-sync binary/container
* Configure your nebula-sync env settings
* Run nebula-sync (on a schedule)

### Configure PiHole instances

The PiHole 6 API has been rewritten and now uses REST. The old API Key has been replaced with an "app password".

To create this password go to Settings -> Web Interface/API -> Advanced Settings (Expert mode must be toggled on in the top right corner to see this section). Click the button to create an app password. Take note of the auto generated string (in Notepad or a password manager) and click again to set the password.

Move to the secondary PiHole instance and repeat.

In addition to making the app password, any secondary PiHole instances must have a setting called `webserver.api.app_sudo` enabled. This setting allows configuration changes to be made through the API. This is disabled by default. This is not needed on the primary instance.

On the secondary instance, go to Settings -> All Settings (make sure you still have expert mode enabled). On the top right, make sure you are set to "All Settings" mode and not "Modified Settings" mode. From here select the Webserver and API section and find the `webserver.api.app_sudo` box. Check off Enabled and click Save & Apply. 

### Download and configure Nebula-sync

Since Nebula-sync uses the API of each configured PiHole instance, it can be hosted anywhere so long as nebula-sync can reach both PiHole instances. I will be hosting mine in its own LXC container.

You can download the latest binary from Github [here](https://github.com/lovelaze/nebula-sync/releases/) using `wget`.

Once you have downloaded the latest release, unpack it with `tar -vxzf nebula-sync_*.tar.gz`. Move the binary to wherever you want it.

Now we can create our configuration file. I'll put it in the same spot as our binary. You can name it whatever you'd like. I'll simply name it `env`.

`nano /opt/nebula-sync/env` 

```bash
PRIMARY=https://primary-pihole|app_password
REPLICAS=https://secondary-pihole|app_password
FULL_SYNC=false
RUN_GRAVITY=false
CLIENT_SKIP_TLS_VERIFICATION=true
#CRON=0 */12 * * *

TZ=America/New_York

SYNC_CONFIG_DNS=true
SYNC_CONFIG_DHCP=true
SYNC_CONFIG_NTP=true
SYNC_CONFIG_RESOLVER=true
SYNC_CONFIG_DATABASE=true
SYNC_CONFIG_MISC=true
SYNC_CONFIG_DEBUG=true

SYNC_GRAVITY_DHCP_LEASES=true
SYNC_GRAVITY_GROUP=true
SYNC_GRAVITY_AD_LIST=true
SYNC_GRAVITY_AD_LIST_BY_GROUP=true
SYNC_GRAVITY_DOMAIN_LIST=true
SYNC_GRAVITY_DOMAIN_LIST_BY_GROUP=true
SYNC_GRAVITY_CLIENT=true
SYNC_GRAVITY_CLIENT_BY_GROUP=true
```

The only lines to change are the `PRIMARY` and `REPLICAS`. You should replace it with your pihole IP addresses and their corresponding app password.

A few things we changed from the example env file provided are `RUN_GRAVITY=false` and `CLIENT_SKIP_TLS_VERIFICATION=true`. Since I use the default, self-signed certificates that came with PiHole, we need to skip the TLS Verification. You can set this back to false if you use real certificates. There seems to be a bug with the `RUN_GRAVITY` option that crashes FTL when nebula-sync tries to request an update to gravity - so we will disable it until it is resolved.

### Run Nebula-sync on a schedule

Unless you have the `CRON` environment set (which only works with the containerized version) - nebula-sync will run once, sync the replicas with the primary, and exit. To schedule nebula-sync, we will use CRON at the system level.

`crontab -e`

Add in the following line to your crontab changing the paths to your env file and binary: `0 * * * * /opt/nebula-sync/nebula-sync run --env-file /opt/nebula-sync/env`

## Closing

With this setup, you should be able to survive one PiHole software or hardware failure and continue on as normal.

I have my primary PiHole instance as a container on my hypervisor, and my secondary instance on a Raspberry Pi 0 W. This allows me to work on and/or restart the hypervisor and have DNS instantly failover to the Pi with no disruptions for clients.
