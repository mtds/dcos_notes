# DC/OS

## What is DC/OS?

DC/OS (the datacenter operating system) is an open-source, distributed operating system based on the Apache Mesos distributed systems kernel. DC/OS manages multiple machines in the cloud or on-premises from a single interface; deploys containers, distributed services, and legacy applications into those machines; and provides networking, service discovery and resource management to keep the services running and communicating with each other.

## Open source or Enterprise?

Main differences between the open source and the enterprise version are highlighted [here](https://mesosphere.com/pricing/).

Summary: in the open source version, the following features **are not available**:
- Non-Disruptive In-Place Upgrade for Kubernetes.
- In-Place Upgrade, Transport Encryption and Kerberos/LDAP Integration.
- High Performance L4/L7 Ingress Load Balancer (Edge-LB).
- Validated DC/OS Upgrades with Automated Pre and Post Upgrade Health Checks.
- Multi-tenancy, security and compliance:
    - No RBAC and Security Audit logging.
    - No Identity Management Integration (Active Directory, LDAP, etc.).
    - No Secrets Management (Key/Value and File-based).
    - No Public Key Infrastructure w/ Custom CA.
- Support for emergency patching.

A complete diagram of the __DC/OS components__, which also highlights the difference between the enterprise and the open source version, is available [here](https://docs.mesosphere.com/1.11/overview/architecture/components/).

## Nodes distribution

Node         | Description
-------------|-------------------------------
lxcm02       | DC/OS bootstrap node
lxzk0[1,2,3] | Master/Zookeeper nodes
lxb00[1,2,3] | Slave nodes

In order to avoid troubles with Zookeeper (especially while restarting the master nodes) or be able to deploy jobs managed by the Marathon scheduler it is **highly recommended** to have VMs with a minimum of 8 Gb of RAM. A complete list of requirements is available on the [DC/OS documentation](https://docs.mesosphere.com/1.11/installing/oss/custom/system-requirements/). The list include how to setup Docker correctly with specific settings, how to __isolate__ the directories on the masters or the agents for better I/O performances, in particular for cluster with potentially thousands of nodes.

## Endpoint Services

Node         |       Description    | Link
-------------|----------------------|-----------------
lxzk0[1,2,3] |  Zookeeper Exhibitor | http://10.1.1.49:8181/exhibitor/v1/ui/index.html
lxzk0[1,2,3] |  DC/OS Web Dashboard | http://10.1.1.49

__Note__: the DC/OS dashboard and the Zookeeper Exhibitor are reachable on all the master nodes.

## Installation

According to ['System Requirements'](https://docs.mesosphere.com/1.11/installing/oss/custom/system-requirements/), all the nodes part of the Mesos cluster should have the following prerequisites:

* Docker must have been installed **before** starting to deploy DC/OS.
* __SELinux__ has to be disabled or set to __permissive__ mode.
* On RHEL 7 and CentOS 7, __firewalld__ must be stopped and disabled.
* __NTP__ support has to be enabled (traditional ntp client or chronyd are equivalent).

Follow the steps described [here ('Advanced DCOS installation procedure for the open source version)'](https://docs.mesosphere.com/1.11/installing/oss/custom/advanced/): there will be a __bootstrap__ node, which will be used to jumpstart the installation of the nodes on the cluster (master or agents). Additional examples for the Mesos cluster configuration file are available [here](https://docs.mesosphere.com/1.11/installing/ent/custom/configuration/examples/).

Apart from the ``cluster.yaml``, which contains the configuration of the DC/OS cluster, the other important piece is the ``ip-detect`` script:  it reports the IP address of each node across the cluster. Each node in a DC/OS cluster has a unique IP address that is used to communicate between nodes in the cluster. The IP detect script prints the unique IPv4 address of a node to STDOUT each time DC/OS is started on the node. There are different ways to gather these IPs: the script can use the AWS or GCE metadata servers, be juse a shell script, etc. The advanced DC/OS installation guide reports all the approach.

__Start to configure the bootstrap node__: use the files under the ``genconf`` subdirectory on this repo:

1. Login as root and create a ``genconf`` subdirectory (e.g. under ``/root``).
2. Download the DC/OS installer: ``curl -O https://downloads.dcos.io/dcos/stable/dcos_generate_config.sh``
3. Launch the installer: ``bash dcos_generate_config.sh`` (**NOTE**: Docker should be already installed and running before this step).
4. Run the Docker container which will use NGINX to serve the DC/OS installation:  
``docker run -d -p 8080:80 -v $PWD/genconf/serve:/usr/share/nginx/html:ro nginx``

Refer to the [documentation](https://docs.mesosphere.com/1.11/installing/oss/custom/configuration/configuration-parameters) to get an idea about the parameters used in the ``genconf/cluster.yaml``.  
**NOTE**: one or more wrongly configured paramters will affect the correct functioning of the master or the agent nodes. In this case the nodes have to be **wiped** and the procedure to create and serve the DC/OS components from the bootstrap node has to be __restarted from scratch__.

Start the installation procedure on a node which is meant to join the cluster (broken down in three steps):

- Install dependencies for the DCOS installed and then install and start Docker:

```bash
>>> yum -y install unzip.x86_64 ; \
    yum -y install bind-utils.x86_64; \
    yum -y install yum-utils device-mapper-persistent-data lvm2 ; \
    yum-config-manager --add-repo  https://download.docker.com/linux/centos/docker-ce.repo ; \ 
    systemctl stop firewalld.service && systemctl disable firewalld.service; \ 
    yum -y install docker-ce ; systemctl enable docker.service && systemctl start docker.service
```

IPtables status after Docker is installed and firewalld is disabled (output adjusted to be more clear):
```bash
>>> iptables -VNL
Chain INPUT (policy ACCEPT 29 packets, 2795 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target                    prot opt in     out        source               destination         
    0     0 DOCKER-USER               all  --  *      *          0.0.0.0/0            0.0.0.0/0           
    0     0 DOCKER-ISOLATION-STAGE-1  all  --  *      *          0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT                    all  --  *      docker0    0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER                    all  --  *      docker0    0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT                    all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 ACCEPT                    all  --  docker0 docker0   0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT 16 packets, 1664 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain DOCKER (1 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER-ISOLATION-STAGE-2  all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DROP       all  --  *      docker0  0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-USER (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

**NOTE**: firewall issues may hamper the installation process and stop services from starting or communicating with peer nodes (e.g. Zookeeper).

Additional information about DC/OS networking is available in the [docs (networking mode, load balancing, etc.)](https://docs.mesosphere.com/1.11/networking/).

- Start the installer from the DCOS bootstrap node:

```bash
>>> groupadd nogroup; mkdir /tmp/dcos && cd /tmp/dcos ; \
    curl -O http://10.1.1.8:8080/dcos_install.sh; \
    bash dcos_install.sh master # or slave
```

Output (master):
```
Starting DC/OS Install Process
Running preflight checks
Checking if DC/OS is already installed: PASS (Not installed)
PASS Is SELinux disabled?
Checking if docker is installed and in PATH: PASS 
Checking docker version requirement (>= 1.6): PASS (18.03.1-ce)
Checking if curl is installed and in PATH: PASS 
Checking if bash is installed and in PATH: PASS 
Checking if ping is installed and in PATH: PASS 
Checking if tar is installed and in PATH: PASS 
Checking if xz is installed and in PATH: PASS 
Checking if unzip is installed and in PATH: PASS 
Checking if ipset is installed and in PATH: PASS 
Checking if systemd-notify is installed and in PATH: PASS 
Checking if systemd is installed and in PATH: PASS 
Checking systemd version requirement (>= 200): PASS (219)
Checking if group 'nogroup' exists: PASS 
Checking if port 53 (required by dcos-net) is in use: PASS 
Checking if port 80 (required by adminrouter) is in use: PASS 
Checking if port 443 (required by adminrouter) is in use: PASS 
Checking if port 1050 (required by dcos-diagnostics) is in use: PASS 
Checking if port 2181 (required by zookeeper) is in use: PASS 
Checking if port 5050 (required by mesos-master) is in use: PASS 
Checking if port 7070 (required by cosmos) is in use: PASS 
Checking if port 8080 (required by marathon) is in use: PASS 
Checking if port 8101 (required by dcos-oauth) is in use: PASS 
Checking if port 8123 (required by mesos-dns) is in use: PASS 
Checking if port 8181 (required by exhibitor) is in use: PASS 
Checking if port 9000 (required by metronome) is in use: PASS 
Checking if port 9942 (required by metronome) is in use: PASS
Checking if port 9990 (required by cosmos) is in use: PASS 
Checking if port 15055 (required by dcos-history) is in use: PASS 
Checking if port 36771 (required by marathon) is in use: PASS 
Checking if port 41281 (required by zookeeper) is in use: PASS 
Checking if port 46839 (required by metronome) is in use: PASS 
Checking if port 61053 (required by mesos-dns) is in use: PASS 
Checking if port 61091 (required by dcos-metrics) is in use: PASS 
Checking if port 61420 (required by dcos-net) is in use: PASS 
Checking if port 62080 (required by dcos-net) is in use: PASS 
Checking if port 62501 (required by dcos-net) is in use: PASS 
Checking Docker is configured with a production storage driver: PASS (overlay2)
Creating directories under /etc/mesosphere
Creating role file for master
Configuring DC/OS   
Setting and starting DC/OS
Created symlink from /etc/systemd/system/multi-user.target.wants/dcos-setup.service to /etc/systemd/system/dcos-setup.service. 
```

Output (slave):
```
Running preflight checks
Checking if DC/OS is already installed: PASS (Not installed)
PASS Is SELinux disabled?
Checking if docker is installed and in PATH: PASS 
Checking docker version requirement (>= 1.6): PASS (18.03.1-ce)
Checking if curl is installed and in PATH: PASS 
Checking if bash is installed and in PATH: PASS 
Checking if ping is installed and in PATH: PASS 
Checking if tar is installed and in PATH: PASS 
Checking if xz is installed and in PATH: PASS 
Checking if unzip is installed and in PATH: PASS 
Checking if ipset is installed and in PATH: PASS 
Checking if systemd-notify is installed and in PATH: PASS 
Checking if systemd is installed and in PATH: PASS 
Checking systemd version requirement (>= 200): PASS (219)
Checking if group 'nogroup' exists: PASS 
Checking if port 53 (required by dcos-net) is in use: PASS 
Checking if port 5051 (required by mesos-agent) is in use: PASS 
Checking if port 61001 (required by agent-adminrouter) is in use: PASS 
Checking if port 61091 (required by dcos-metrics) is in use: PASS 
Checking if port 61420 (required by dcos-net) is in use: PASS 
Checking if port 62080 (required by dcos-net) is in use: PASS 
Checking if port 62501 (required by dcos-net) is in use: PASS 
Checking Docker is configured with a production storage driver: PASS (overlay2)
Creating directories under /etc/mesosphere
Creating role file for slave
Configuring DC/OS
Setting and starting DC/OS
Created symlink from /etc/systemd/system/multi-user.target.wants/dcos-setup.service to /etc/systemd/system/dcos-setup.service.
```

## Operations

Check all the service components on the master nodes:
```bash
>>> journalctl -u dcos-exhibitor -b
[...]
>>> journalctl -u dcos-mesos-master -b
[...]
>>> journalctl -u dcos-mesos-dns -b
[...]
>>> journalctl -u dcos-marathon -b
[...]
>>> journalctl -u dcos-nginx -b
[...]
>>> journalctl -u dcos-gen-resolvconf -b
[...]
```

Check all the service components on the slave nodes:
```bash
>>> journalctl -u dcos-mesos-slave -b
[...]
```

## Troubleshooting

References:
- [DC/OS troubleshooting docs (Open Source version)](https://docs.mesosphere.com/1.11/installing/oss/troubleshooting/)
- [DC/OS troubleshooting (DC/OS official channel on YouTube)](https://www.youtube.com/watch?v=YB2N5COpWY4)

## Web Dashboards

The following image shows the main page of a DC/OS dashboard (while a bunch of tasks are running):

![DCOS dashboard](/imgs/DCOS_Dashboard.png)


The following image shows the Exhibitor web page for Zookeeper (available only when the cluster is up&running):

![Zookeeper Exhibitor](/imgs/Zookeeper_Exhibitor.png)

## Uninstall DC/OS

According to the [official documentation](https://docs.mesosphere.com/1.11/installing/oss/custom/uninstall/): 'To remove DC/OS, you __must completely reimage the operating system on your nodes__. Uninstall will be supported in future releases.' For more information, see the following ticket: ['Create a comprehensive DC/OS uninstall'](https://jira.mesosphere.com/browse/DCOS_OSS-250).

## References

* [DCOS main website](https://dcos.io/)
* [DCOS documentation](https://docs.mesosphere.com/1.11/)
