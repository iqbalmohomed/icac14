# Runbook to setup devstack with neutron using Vagrant

## Summary of provisioning done in Vagrantfile

The Vagrantfile is setup to give the devstack VM a single network interface. We assume the IP is 10.0.2.15.

As part of the provisioning process of the vagrant vm, we:

1. Create a user called stack with a weak password (**password**)
2. Install required packages (git, bridge-utils)
3. Create a bridge called br0, attach eth0 to this bridge and have the bridge get its IP address via DHCP (which should be 10.0.2.15)
4. Clone the devstack repo (we checkout the icehouse branch)

We start as usual:

vagrant up

## Running stack.sh
Once the vagrant provisioning process is done, log into the vm and run stack.sh as usual:

vagrant ssh

sudo su stack

cd /home/stack/devstack

./stack.sh

## Fix bridging

We must remove an extra route created by the devstack installation:

sudo route

sudo route del -net 192.168.15.0 netmask 255.255.255.224 br-ex

## Add DNS to private network and optionally open ports for ICMP, ssh and web

source openrc admin admin

neutron subnet-update private-subnet --dns_nameservers list=true 8.8.8.8 8.8.4.4

The following three commands may not be necessary. If after booting the VM, you cannot ping or log into it after a minute, then try these out:

neutron security-group-rule-create --protocol icmp --direction ingress default

neutron security-group-rule-create --protocol tcp --port-range-min 22 --port-range-max 22 --direction ingress default

neutron security-group-rule-create --protocol tcp --port-range-min 80 --port-range-max 80 --direction ingress default

## Setup usual Openstack resources

We setup a keypair and the default security group to allow ICMP and ssh access

### Setup keys

ssh-keygen -t rsa -f cloud

nova keypair-list

nova keypair-add --pub-key cloud.pub cloud

### Security Groups

nova secgroup-list

nova secgroup-list-rules default

nova secgroup-add-rule default tcp 22 22 0.0.0.0/0

nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0

## Create new port and boot VM

Finally, we create a port on the private subnet and boot up a vm. I give an example with the IDs generated on my system. You should substitute with the ones generated on your installation:

Make a note of the private network's ID in both of the following commands:

nova net-list

neutron subnet-list

In the following, the first ID is the subnet ID and the second is the network id:

neutron port-create --fixed-ip subnet_id=c57a0503-15c9-44a6-8b4c-24c2b48285b5,ip_address=10.0.0.4 88abd80a-4fc3-4421-8c1d-9617648a2209

In the following, the first ID is the image id (nova image-list) and the second id is from the port created in the previous line.

nova boot --image b62280fe-c3a2-464d-bde6-f989b301789f --flavor m1.nano --key-name cloud --nic port-id=1c70d7f1-9c36-4bb6-86d2-d48ed7e70019 newvm

sudo ip netns list

You should be able to ping the VM using the network namespace corresponding to the router:

sudo ip netns exec qrouter-16b05e53-900f-4cf2-b7d4-18033555bbdc ping 10.0.0.4

You should also be able to ssh into it:

sudo ip netns exec qrouter-16b05e53-900f-4cf2-b7d4-18033555bbdc ssh cirros@10.0.0.4

Note 1: the password for these images is cubswin:)

Note 2: the vms should be able to connect to the outside world.
