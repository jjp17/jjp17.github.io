---
title: "Setting up Keepalived in the cloud - AWS"
date: 2025-12-09T23:30:00-05:00
# weight: 1
# aliases: ["/first"]
tags: ["aws", "cloud", "keepalived"]
categories: ["Selfhosting"]
#author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "An example setup of Keepalived in AWS"
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

Keepalived provides frameworks for both load balancing and high availability. Unless you have a specific use case for keepalived in the cloud, there are almost always better methods for load balancing or HA. This article is just to prove it is possible to use in the cloud and a creative way I've utilized it.

## What makes keepalived in the cloud different?

There are two main issues with using keepalived in the cloud. VRRP and the VIP.

### VRRP

VRRP is the main protocol that Keepalived uses to communicate with your "cluster". 

VRRP stands for Virtual Router Redundancy Protocol. The cluster will determine which machine is the "master" based on the priority each machine advertises. A priority of 255 is the highest, while a priority of 1 is the lowest.
When the master machine is determined, it will periodically send advertisements to the backup machines to let them know it is healthy. 

The issue? VRRP and other multicast protocols don't work well (if at all) inside of AWS VPCs. 

We can solve this by switching our keepalived config to use unicast and specifying each of the other hosts in the "cluster". 

### Moving the Virtual IP to another host

In a regular network it is easy to pass a VIP between hosts. In cloud environments, admittedly for reasons I don't fully understand, this is much more difficult (if not impossible) to do.

The solution to this is to write a script that lifts and shifts a network interface from one machine to another using AWS CLI.

## A creative use case

The original use case for this was to help cut cloud costs by shutting down application servers when idle and then allowing application users (without access to AWS) to turn the servers back on when needed.

When users went to the webUI, they were coming in from the virtual IP and accessing their application normally. If it was determined that the application was not being used, all the relevant instances would be shutdown automatically. The virtual IP would be passed to the cheapest possible EC2 instance that was always on, who's entire purpose was to display a basic web page with a button that reads **"Turn my application on"**

Clicking that button would trigger a call to AWS CLI to turn the application servers back on. 

After a few minutes, once all the servers were up again, the VIP would be passed back to the main web server and the page would refresh, bringing the user back to their familiar WebUI.

## Keepalived config

I will outline an example using two EC2 instances. Just one for the master and one for the backup. 

Keepalived can be installed with your distro package manager.

### Master instance config

Configurations on both nodes will be very standard. The only major difference will be the addition of the addition of the unicast option.

```bash
sudo touch /etc/keepalived/keepalived.conf
sudo nano /etc/keepalived/keepalived.conf
```

We will use the following config file:

```bash
vrrp_instance ha_1 {
        state MASTER
        interface ens5
        virtual_router_id 250
        priority 255
        advert_int 1

        notify /etc/keepalived/scripts/notify.sh

        unicast_src_ip <Master node IP> 
        unicast_peer {
                <Backup node IP>
        } 

        authentication {
                auth_type PASS
                auth_pass <your cluster password>
        }  
}
```

Notice the `unicast_src_ip` line and `unicast_peer` block above.

We will also need a keepalived notify script.

```bash
sudo touch /etc/keepalived/scripts/notify.sh
sudo nano /etc/keepalived/scripts/notify.sh
```

We will use this notify config file:

```bash
#!/bin/bash

TYPE=$1
NAME=$2
STATE=$3

case $STATE in
        "MASTER") /etc/keepalived/scripts/master.sh
                  exit 0
                  ;;
        "BACKUP") echo "Backup"
                  exit 0
                  ;;
        "FAULT")  echo "Fault"
                  exit 0
                  ;;
        *)        echo "Unknown State"
                  exit 1
                  ;;
esac
```

This is a pretty standard example of a notify script. There is another script we will have to write that the host will execute when it becomes the master of the cluster. 

### Backup instance config

Configuration on the backup instance is identical to the master with two exceptions. The priority of the backup should be lower than the master. The `unicast_src` should be switched to the IP of the backup node and the `unicast_peer` should be the IP of the master node. 

The backup instance config file will look like this:

```bash
vrrp_instance ha_1 {
        state BACKUP
        interface ens5
        virtual_router_id 250
        priority 254
        advert_int 1

        notify /etc/keepalived/scripts/notify.sh

        unicast_src_ip <Backup node IP> 
        unicast_peer {
                <Master node IP>
        } 

        authentication {
                auth_type PASS
                auth_pass <your cluster password>
        }  
}
```

The notify script will be identical to the masters

## Modifications within AWS

### Granting AWS CLI permissions to your machines

For our solution to work we will need to make calls using AWS CLI to search for, attach, and detach network interfaces from our machines. 

Both your Master and Backup node will need permissions to perform the following AWS CLI calls:

* `aws ec2 describe-network-interface`
* `aws ec2 detach-network-interface`
* `aws ec2 attach-network-interface`

### Creating a Floating NIC

In place of moving a virtual IP address between machines with keepalived, we will be physically detaching a network interface from the old master instance and attaching it to the new master instance.

Create a new Network Interface in AWS. Make sure the `Name` tag is populated as this will be how we locate the proper NIC in our script.

## The master notify script

The master notify script will always run on the new master instance. This script will find our floating NIC we created, detach it from the old master instance, reattach it to the new (current) master instance, bring up the new interface, and restart any applications.

This script will need to be created on both Master and Backup instances.

```bash
sudo touch /etc/keepalived/scripts/master.sh
sudo nano /etc/keepalived/scripts/master.sh
```

```bash
#!/bin/bash

#Vars
nic_name="Your NICs name tag here"

#Find the Elastic Network Interface
networkInterfaceId=$(aws ec2 describe-network-interfaces --filters Name=tag:Name,Values="${nic_name}" | jq -r .NetworkInterfaces[0].NetworkInterfaceId)
echo "Found networkInterfaceID: $networkInterfaceId"
#Find our NICs attachment ID
attachmentId=$(aws ec2 describe-network-interfaces --filters Name=tag:Name,Values="${nic_name}" | jq -r .NetworkInterfaces[0].Attachment.AttachmentId)
echo "Found attachmentID: $attachmentId"
#Detach network interface from our OLD MASTER instance using our attachment ID
aws ec2 detach-network-interface --attachment-id $AttachmentId

#Ensure our NIC is available
state="none"
until [[state == "available" ]]
do
    state=$(aws ec2 describe-network-interfaces --filters Name=tag:Name,Values="${nic_name}" | jq -r .NetworkInterfaces[0].Status)
    echo "NIC is currently: $state"
    if [[ $state == "available" ]]
        then
        break;
    fi
    sleep 2
done

#Find instance ID of our NEW master
instanceId=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.instanceId')
echo "Found instanceID: $instanceId"

#Attach the NIC to our NEW master
aws ec2 attach-network-interface --network-interface-id $networkInterfaceId

#Check if we're running Amazon Linux first since AL will automatically bring up the interface
source /etc/os-release
if [ "$NAME" != "Amazon Linux" ]; then
    echo "Bringing up interface"
    
    #Check to see if our NIC is attached properly
    state="1"
    until [[ $state -eq "0" ]]
    do
        ip -br a | grep eth1
        state=$1
        echo "NIC is currently: $state"
        if [[ $state -eq "0" ]]; then
            break
        fi
        sleep 1
    done

    #Bring new NIC down and back up again
    ip link set dev eth1 down
    kill $(ps aux | grep "[d]hclient eth1" | awk '{ print $2 }')
    /sbin/dhclient eth1
fi

#Restart your relevant applications
systemctl restart nginx.service
```

Granted - it's a bit hacky but it works. You may need to modify what the link is named in your OS (eth1, ens0, etc) or change the method to bring the new interface up.

## Going further

At this point we should have a fully functioning Master/Backup setup with Keepalived in AWS. When the master instance goes offline, the backup will take control of the floating Network Interface and attach it to itself, simulating the action of moving a VIP.

For my use case described above, we would make another web server on the backup instance and write another script to turn on the main instances with a button displayed on the backup webUI. You'd probably want to add some authentication layer in front as well (unless you everybody to turn your instances back on)

## Closing

Is this recommended to use this method in the cloud? No.

Are there better ways to do this? Probably.

This is the solution I came up with on the fly for saving some money. It serves its purpose and shows how keepalived can theoretically be used in the cloud. 