# Alta Disponibilidade
### Corosync + Pacemaker

*Corosync* is an open source program that provides cluster membership and messaging capabilities, often referred to as the messaging layer, to client servers.

*Pacemaker* is an open source cluster resource manager (CRM), a system that coordinates resources and services that are managed and made highly available by a cluster. In essence, Corosync enables servers to communicate as a cluster, while Pacemaker provides the ability to control how the cluster behaves.

--- 

#### Synchronizing time betweenservers
Whenever you have multiple servers communicating with each other, especially with clustering software, it is important to ensure their clocks are synchronized. Let’s use NTP (Network Time Protocol) to synchronize our servers. On two servers, run those commands, select the same timezone on both servers:

```
sudo dpkg-reconfigure tzdata
sudo apt-get update
sudo apt-get -y install ntp
```

#### Configure Firewall
Corosync uses UDP transport between ports 5404, 5405 and 5406 . If you are running a firewall, ensure that communication on those ports are allowed between the servers.

If you use ufw, you could allow traffic on these ports with these commands on both servers:

`sudo ufw allow 5404, 5405, 5406`

Or if you use iptables, you could allow traffic on these ports and eth1 (the private network interface) with these commands:

```
sudo iptables -A INPUT  -i eth1 -p udp -m multiport --dports 5404,5405,5406 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT  -o eth1 -p udp -m multiport --sports 5404,5405,5406 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

#### Install Corosync and Pacemaker
Corosync is a dependency of Pacemaker, so we can install both of them using one command. Run this command on both servers:

`sudo apt-get install pacemaker`

#### Configure Authorization Key for two servers
Corosync must be configured so that our servers can communicate as a cluster.

On server A (main server), run these commands:

```
sudo apt-get install haveged
sudo corosync-keygen
```

This will generate a 128-byte cluster authorization key, and write it to /etc/corosync/authkey on server A. Now we need to run this command on server A to copy the authkey to server B (backup server)

`sudo scp /etc/corosync/authkey username@server_B_ip:  /etc/corosync/`

Then, on server B, run thoses commands:

```
sudo chown root: /etc/corosync/authkey
sudo chmod 400 /etc/corosync/authkey
```

#### Configure Corosync cluster
On both servers, open the corosync.conf and write the below scripts:

```
totem {
  version: 2
  cluster_name: lbcluster
  transport: udpu
  interface {
    ringnumber: 0
    bindnetaddr: private_binding_IP_address
    broadcast: yes
    mcastport: 5405
  }
}
quorum {
  provider: corosync_votequorum
  two_node: 1
}
nodelist {
  node {
    ring0_addr: server_A_private_IP_address
    name: primary
    nodeid: 1
  }
  node {
    ring0_addr: server_B_private_IP_address
    name: secondary
    nodeid: 2
  }
}
logging {
  to_logfile: yes
  logfile: /var/log/corosync/corosync.log
  to_syslog: yes
  timestamp: on
}
```

Para usar multiplos clusters na mesma rede, é necessário alterar o endereço de socket do corosync que é representado pela configuração `mcastaddr: 192.168.0.10`.

exemplo em produção: 
```
totem {
  version: 2
  cluster_name: lbcluster
  transport: udpu
  interface {
    ringnumber: 0
    bindnetaddr: 192.168.0.0
    broadcast: yes
    mcastaddr: 192.168.0.10
    mcastport: 5405
  }
}
quorum {
  provider: corosync_votequorum
  two_node: 1
}
nodelist {
  node {
    ring0_addr: 192.168.0.1
    name: primary
    nodeid: 1
  }
  node {
    ring0_addr: 192.168.0.2
    name: secondary
    nodeid: 2
  }
}
logging {
  to_logfile: yes
  logfile: /var/log/corosync/corosync.log
  to_syslog: yes
  timestamp: on
}

```

You can try to read the scripts and try to understand it. If you can’t, just forget about it :). There are only something that’s you need to remember:

- *server_A_private_IP_address*: Private IP of server A
- *server_B_private_IP_address*: Private IP of server B

- *private_binding_IP_address*: The private IP that’s both server A and B are binding to). To know this address, just run ifconfig on server A (or server B) and take a look at the private interface (usually eth1), you will see something like below, the IP 2.0.0.255 is the value for private_binding_IP_address, because 2 server are running in the same private network, this value must be the same on both server:

`inet addr:2.0.0.1 Bcast:2.0.0.255  Mask:255.255.255.0`

#### Enable and run Corosync
Next, we need to configure Corosync to allow the Pacemaker service. On both servers, create the pcmk file in the Corosync’s service directory with below commands:

```
sudo mkdir -p /etc/corosync/service.d
sudo vim /etc/corosync/service.d/pcmk
```

Then add this scripts to the pcmkfile

```
service {
  name: pacemaker
  ver: 1
}
```

Finally, open file /etc/default/corosync and add this line (if there is already a line START=no, change it to YES as below)

`START=yes`


Now, start Corosync on both server

`sudo service corosync start`

Let’s check if everything is working ok with command:

`sudo corosync-cmapctl | grep members`

This should output something like this (if not, wait 1 minute and run the command again):

```
runtime.totem.pg.mrp.srp.members.1.config_version (u64) = 0
runtime.totem.pg.mrp.srp.members.1.ip (str) = r(0) ip(server_A_private_IP_address)
runtime.totem.pg.mrp.srp.members.1.join_count (u32) = 1
runtime.totem.pg.mrp.srp.members.1.status (str) = joined
runtime.totem.pg.mrp.srp.members.2.config_version (u64) = 0
runtime.totem.pg.mrp.srp.members.2.ip (str) = r(0) ip(server_B_private_IP_address)
runtime.totem.pg.mrp.srp.members.2.join_count (u32) = 1
runtime.totem.pg.mrp.srp.members.2.status (str) = joined
```

#### Enable and Start Pacemaker
Pacemaker, which depends on the messaging capabilities of Corosync, is now ready to be started. On both servers, enable Pacemaker to start on system boot with this command:

`sudo update-rc.d pacemaker defaults 20 01`

Because Pacemaker need to start after Corosync, we set Pacemaker’s start priority to 20, which is higher than Corosync's (it’s 19 by default).

Now let’s start Pacemaker:

`sudo service pacemaker start`

To interact with Pacemaker, we will use the crm utility. Check Pacemaker’s status:

`sudo crm status`

This should output something like this (if not, wait for 30 seconds and run the command again):

```
Last updated: Sun Sep 17 15:49:24 2017          
Last change: Tue Sep 12 09:04:23 2017 by root via crm_attribute on secondary
Stack: corosync
Current DC: primary (version 1.1.14-70404b0) - partition with quorum
2 nodes and 0 resource configured
Online: [ primary secondary ]
```

### Configure Pacemaker and add our Public IP as a Resource

First we need to config some properties. We can run Pacemaker (crm) commands from either server, as it automatically synchronizes all cluster-related changes across all member nodes. Let’s try to run those commands on server A

```
sudo crm configure property stonith-enabled=false
sudo crm configure property no-quorum-policy=ignore
```

Now we will add our public IP (1.0.0.3) as a Resource with this command:

```
sudo crm configure primitive virtual_public_ip \
ocf:heartbeat:IPaddr2 params ip="1.0.0.3" \
cidr_netmask="32" op monitor interval="10s" \
meta migration-threshold="2" failure-timeout="60s" resource-stickiness="100"
```

NOTE: The config resource-stickiness=”100" means that’s whenever a server take the resource, our public IP (1.0.0.3), because the other server is down, it will take it forever even when the other server is online again.

Check the Pacemaker’s status again with command ‘sudo crm status’ you can see:

```
Last updated: Sun Sep 17 15:49:24 2017          
Last change: Tue Sep 12 09:04:23 2017 by root via crm_attribute on secondary
Stack: corosync
Current DC: primary (version 1.1.14-70404b0) - partition with quorum
2 nodes and 1 resource configured
Online: [ primary secondary ]
Full list of resources:
virtual_public_ip   (ocf::heartbeat:IPaddr2):    Started primary
```

So we are having one resource running and the primary node (server A) is taking it. It means server A is handle our public IP (1.0.0.3). To double check this, try to run command:

`ip -4 addr ls`

You should see:

```
...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 1.0.0.1/24 brd 1.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 1.0.0.3/32 brd 1.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
...
```

Testing, simulate the situation when server A going down
Now, we try to simulate the situation when server A is down, server B should take the public IP (1.0.0.3) in this case.

Of course you can shutdown server A, but if you really don’t want to shut it down, you can make the primary node become standby with command:

`sudo crm node standby primary`

Let’s open server B and check pacemaker status with command ‘sudo crm status’ you should see:

```
Last updated: Sun Sep 17 15:49:24 2017          
Last change: Tue Sep 12 09:04:23 2017 by root via crm_attribute on secondary
Stack: corosync
Current DC: primary (version 1.1.14-70404b0) - partition with quorum
2 nodes and 1 resource configured
Node primary: standby
Online: [ secondary ]
Full list of resources:
virtual_public_ip   (ocf::heartbeat:IPaddr2):    Started secondary
```

Check the server B’s ip with:

`ip -4 addr ls`

You should see server B is now taking our public IP:

```
...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 1.0.0.2/24 brd 1.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 1.0.0.3/32 brd 1.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
...
```

Now, to make the server A online again:

`sudo crm node online primary`

Because we set the resource-stickiness=”100" we need to make secondary node standby and online again to make primary node take our public IP again as default setting

```
sudo crm node standby secondary
sudo crm node online secondary
```



