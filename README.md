# DC/OS

Scope of this document is to create a cluster of VMs (KVM/LibVirt based), which is able to run [Mesosphere DC/OS](https://mesosphere.com/).

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

All the VMs are created with the [vm-tools](https://github.com/vpenso/vm-tools) toolchain.

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

Apart from the ``cluster.yaml``, which contains the configuration of the DC/OS cluster, the other important piece is the ``ip-detect`` script:  it reports the IP address of each node across the cluster. Each node in a DC/OS cluster has a unique IP address that is used to communicate between nodes in the cluster. The IP detect script prints the unique IPv4 address of a node to STDOUT each time DC/OS is started on the node. There are different ways to gather these IPs: the script can use the AWS or GCE metadata servers, be juse a shell script, etc. The advanced DC/OS installation guide reports all the approaches.

__Start to configure the bootstrap node__: use the files under the ``genconf`` subdirectory on this repo:

1. Login as root and create a ``genconf`` subdirectory (e.g. under ``/root``).
2. Download the DC/OS installer: ``curl -O https://downloads.dcos.io/dcos/stable/dcos_generate_config.sh``
3. Launch the installer: ``bash dcos_generate_config.sh`` (**NOTE**: Docker should be already installed and running before this step).
4. Run the Docker container which will use NGINX to serve the DC/OS installation:  
``docker run -d -p 8080:80 -v $PWD/genconf/serve:/usr/share/nginx/html:ro nginx``

Refer to the [documentation](https://docs.mesosphere.com/1.11/installing/oss/custom/configuration/configuration-parameters) to get an idea about the parameters used in the ``genconf/cluster.yaml``.  
**NOTE**: one or more wrongly configured paramters will affect the correct functioning of the master or the agent nodes. In this case the nodes have to be **wiped** and the procedure to create and serve the DC/OS components from the bootstrap node has to be __restarted from scratch__.

__Start the installation procedure on a node which is meant to join the cluster__: it's broken down in three steps.

- Install dependencies for DC/OS and then install and start Docker:

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

- Start the DC/OS installer from the bootstrap node:

```bash
>>> groupadd nogroup; mkdir /tmp/dcos && cd /tmp/dcos ; \
    curl -O http://10.1.1.8:8080/dcos_install.sh; \
    bash dcos_install.sh master # or slave
```

**NOTE**: do not proceed to install the slave nodes until you have a fully functional and responsive Zookeeper cluster.

Output of the installation process (master):
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

Output of the installation process (slave):
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

## DC/OS command line examples

Once the cluster is up and running, it's also possible to interact with DC/OS using the ``dcos`` command line interface, as explained [here](https://docs.mesosphere.com/1.11/cli/). After the installation, a subdirectory called ``.dcos`` will be created under ``$HOME``. **NOTE**: since the deployment scenario in this document keep things as simple as possible, there is no authentication mechanism to interact with the DC/OS dashboard.

1. Connect to your DC/OS virtual cluster: ``dcos cluster setup http://10.1.1.49`` (no output will be reported if successfully connected)
2. Shows information about the cluster:

```bash
>>> dcos cluster list
   NAME                 CLUSTER ID                 STATUS   VERSION        URL
dcos_gsi*  7231c375-2f80-4727-a516-4737d7c253af  AVAILABLE   1.11.2  http://10.1.1.49
```
3. Shows masters and slaves (agents) nodes:

```bash
>>> dcos node
HOSTNAME        IP                        ID                       TYPE             REGION  ZONE  
10.1.1.13      10.1.1.13  9928caa0-c66b-4fc6-8df6-6509034c7299-S0  agent             None   None  
master.mesos.  10.1.1.49                    N/A                    master            N/A    N/A   
master.mesos.  10.1.1.50                    N/A                    master            N/A    N/A   
master.mesos.  10.1.1.51    13c1ba1f-3c71-4f89-8d4d-3387cb367fd5   master (leader)   None   None
```
4. Shows running services on the cluster:

```bash
>>> dcos service
NAME          HOST    ACTIVE  TASKS  CPU  MEM  DISK  ID                                         
marathon   10.1.1.50   True     0    0.0  0.0  0.0   9928caa0-c66b-4fc6-8df6-6509034c7299-0001  
metronome  10.1.1.51   True     0    0.0  0.0  0.0   9928caa0-c66b-4fc6-8df6-6509034c7299-0000
```
5. Shows information about the Marathon scheduler:

```bash
>>> dcos marathon about
{
  "buildref": "f9d087d2fbad410adf512a08196206b302f417fb",
  "elected": true,
  "frameworkId": "9928caa0-c66b-4fc6-8df6-6509034c7299-0001",
  "http_config": {
    "http_port": 8080,
    "https_port": 8443
  },
  "leader": "10.1.1.50:8080",
  "marathon_config": {
    "access_control_allow_origin": null,
    "checkpoint": true,
    "decline_offer_duration": 300000,
    "default_network_name": "dcos",
    "env_vars_prefix": null,
    "executor": "//cmd",
    "failover_timeout": 604800,
    "features": [
      "vips",
      "task_killing",
      "external_volumes",
      "gpu_resources"
    ],
    "framework_name": "marathon",
    "ha": true,
    "hostname": "10.1.1.50",
    "launch_token": 100,
    "launch_token_refresh_interval": 30000,
    "leader_proxy_connection_timeout_ms": 5000,
    "leader_proxy_read_timeout_ms": 10000,
    "local_port_max": 20000,
    "local_port_min": 10000,
    "master": "zk://zk-1.zk:2181,zk-2.zk:2181,zk-3.zk:2181,zk-4.zk:2181,zk-5.zk:2181/mesos",
    "max_instances_per_offer": 100,
    "mesos_bridge_name": "mesos-bridge",
    "mesos_heartbeat_failure_threshold": 5,
    "mesos_heartbeat_interval": 15000,
    "mesos_leader_ui_url": "/mesos",
    "mesos_role": "slave_public",
    "mesos_user": "root",
    "min_revive_offers_interval": 5000,
    "offer_matching_timeout": 3000,
    "on_elected_prepare_timeout": 180000,
    "reconciliation_initial_delay": 15000,
    "reconciliation_interval": 600000,
    "revive_offers_for_new_apps": true,
    "revive_offers_repetitions": 3,
    "scale_apps_initial_delay": 15000,
    "scale_apps_interval": 300000,
    "store_cache": true,
    "task_launch_confirm_timeout": 300000,
    "task_launch_timeout": 86400000,
    "task_lost_expunge_initial_delay": 300000,
    "task_lost_expunge_interval": 30000,
    "task_reservation_timeout": 20000,
    "webui_url": null
  },
  "name": "marathon",
  "version": "1.6.392",
  "zookeeper_config": {
    "zk": "zk://zk-1.zk:2181,zk-2.zk:2181,zk-3.zk:2181,zk-4.zk:2181,zk-5.zk:2181/marathon",
    "zk_compression": true,
    "zk_compression_threshold": 65536,
    "zk_connection_timeout": 10000,
    "zk_max_node_size": 1024000,
    "zk_max_versions": 50,
    "zk_session_timeout": 10000,
    "zk_timeout": 10000
  }
}
```

## Operations

Check all the DC/OS service components on the master nodes:
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

When troubleshooting problems with a DC/OS installation, you should explore the components in this sequence:

1. Exhibitor
2. Mesos master
3. Mesos DNS
4. DNS Forwarder
5. DC/OS Marathon
6. Jobs
7. Admin Router

Be sure to check that all services are up and healthy on the masters **before** checking the agents.

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
* [DCOS Design](https://docs.mesosphere.com/1.11/overview/design/)
* [DCOS Cmd Line interface](https://docs.mesosphere.com/1.11/cli/)
* [DCOS Tutorials about run and operate services on top of it](https://docs.mesosphere.com/1.11/tutorials/)
* [DCOS Frequently Asked Questions](https://docs.mesosphere.com/1.11/installing/oss/faq/)
* [DCOS Monitoring with Prometheus](https://docs.mesosphere.com/1.11/metrics/prometheus/)
* [Building and Running Distributed Systems using Apache Mesos - Benjamin Hindman at ApacheCon NA (2014)](https://www.youtube.com/watch?v=hTcZGODnyf0)
