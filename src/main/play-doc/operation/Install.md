# Installation

Choose on of the following installation guides to get started:

* [Linux Installation](#Linux-Installation)
* [EC2 Installation](#EC2-Installation)
* [DC/OS Installation](#DC/OS-Installation)


# Linux Installation

> In order to obtain the Debian or RPM installations of ConductR then please [contact our sales department](https://www.lightbend.com/company/contact). To evaluate ConductR in general then [please visit our product page](http://www.lightbend.com/products/conductr) which provides instructions on getting started. Otherwise if you are looking to use ConductR for free from a development perspective then please [head over to our developer section](DevQuickStart).

-

> This is a tutorial for installing ConductR on linux in production mode. It shows how this is done for a small cluster of 3 machines. If you are looking for a non-production Linux installation (for example, a QA environment that is close to production), be sure to read about [how to setup for non-production](ClusterSetupConsiderations#Setting-up-for-non-production) after reading the remainder of this page.

## Prerequisites

* x86/64 bit Debian or Rpm based Linux system (recommended: Ubuntu 14.04 LTS or RHEL/CentOS 7)
* Oracle Java Runtime Environment 8 (JRE 8)
* Python 3.4
* Debian or Rpm package of ConductR Core
* Debian or Rpm package of ConductR Agent

### Installing JRE 8

Install Java 8 as the default JRE on the system.

On Ubuntu, you can use the webupd8team repository provided that you accept the Oracle license agreement.

```bash
sudo add-apt-repository -y ppa:webupd8team/java
sudo apt-get update
sudo apt-get -y install oracle-java8-installer && sudo apt-get clean
sudo apt-get -y install oracle-java8-set-default
echo "JAVA_HOME=/usr/lib/jvm/java-8-oracle" | sudo tee -a /etc/environment
```

### Optional dependencies

* Docker - for running bundles inside a Docker container

## Installing ConductR on the first machine

ConductR comprises of the ConductR Core and ConductR Agent. ConductR Core is responsible for cluster-wide scaling and replication decision making, as well as hosting the application files or the bundles. ConductR Agent is responsible for executing the application processes. The ConductR Core and the ConductR Agent are ran as separate processes, and hence separate services within the operating system.

This tutorial uses three systems with the addresses `172.17.0.{1,2,3}`. To simplify the installation and configuration instructions, we are going to use the hostname command. Please ensure the hostname is set correctly or substitute your addresses as appropriate for $(hostname). To set the hostname, pass the ip address to hostname.

```bash
sudo hostname $(hostname -i)
```

### Installing ConductR Core on the first machine

The tutorial also assumes that you have obtained the `conductr_%PLAY_VERSION%_all.deb` Debian or `conductr_%PLAY_VERSION%-1.noarch.rpm` Rpm package.

Install ConductR Core as any other Debian or Rpm package.

```bash
[172.17.0.1]$ sudo dpkg -i conductr_%PLAY_VERSION%_all.deb
```
or

```bash
[172.17.0.1]$ sudo yum install conductr-%PLAY_VERSION%-1.noarch.rpm
```

ConductR Core is automatically registered as a service and started. ConductR provides cluster and application information as well as its control interface via a REST API exposed as part of the ConductR Core.

```bash
[172.17.0.1]$ curl -s 127.0.0.1:9005/members | python3 -m json.tool
```

A typical response contains the current members of the cluster (shown here as just one), the address of the node that the queried control server is running on and a list of unreachable nodes (shown here as empty).

``` json
{
  "members": [
    {
      "node": "akka.tcp://conductr@127.0.0.1:9004",
      "nodeUid": "-1595142725",
      "roles": [
        "web"
      ],
      "status": "Up"
    }
  ],
  "selfNode": "akka.tcp://conductr@127.0.0.1:9004",
  "selfNodeUid": "-1595142725",
  "unreachable": []
}
```

The IP addresses in the response indicate that ConductR Core is listening to the `localhost` address. To be able to form an inter-machine cluster, ConductR Core must be configured to listen to the machine's host interface. This can be enabled adding a property declaration for `CONDUCTR_IP` to the start command as follows:

```bash
[172.17.0.1]$ echo -DCONDUCTR_IP=$(hostname) | sudo tee -a /usr/share/conductr/conf/conductr.ini
[172.17.0.1]$ sudo service conductr restart
```

Check for the cluster information once again, but now use the host address of the machine.

```bash
[172.17.0.1]$ curl -s $(hostname):9005/members | python3 -m json.tool
```

The ConductR Core service runs under the `conductr` user along with the `conductr` group. Its pid file is written to: `/var/run/conductr/running.pid` and its install location is `/usr/share/conductr`.

### Installing ConductR Agent on the first machine

The tutorial also assumes that you have obtained the `conductr-agent_%PLAY_VERSION%_all.deb` Debian or `conductr-agent_%PLAY_VERSION%-1.noarch.rpm` Rpm package.

Install ConductR Agent as any other Debian or Rpm package.

```bash
[172.17.0.1]$ sudo dpkg -i conductr-agent_%PLAY_VERSION%_all.deb
```
or

```bash
[172.17.0.1]$ sudo yum install conductr-agent-%PLAY_VERSION%-1.noarch.rpm
```

ConductR Agent is automatically registered as a service and started.

ConductR Agent needs to be connected to a ConductR core node in order for ConductR to run any application process. To establish this connection, configure ConductR Agent as such:

```bash
[172.17.0.1]$ echo -Dconductr.agent.ip=$(hostname) | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ echo --core-node $(hostname):9004 | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
```

Once configured, restart the ConductR Agent service.

```bash
[172.17.0.1]$ sudo service conductr-agent restart
```

The ConductR Agent service runs under the `conductr-agent` user along with the `conductr-agent` group. Its pid file is written to: `/var/run/conductr-agent/running.pid` and its install location is `/usr/share/conductr-agent`.


### Consolidated Logging

Enabling consolidated logging will require configuration for each machine where ConductR is installed. [[Consolidated logging|ConsolidatedLogging]] section describes the steps required which allow you to select the appropriate logging method for you. ConductR is bundled with an Elasticsearch based solution and is configured for that by default.

By default ConductR's logging is quite sparse. Unless an error or warning occurs then there will be no log output.

To increase the verbosity of the ConductR Core logging you can use this command:

```bash
[172.17.0.1]$ echo -Dakka.loglevel=debug | sudo tee -a /usr/share/conductr/conf/conductr.ini
```

Similarly for ConductR Agent:

```bash
[172.17.0.1]$ echo -Dakka.loglevel=debug | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
```

### Optional dependencies

#### Docker

ConductR supports running applications and services within Docker. If you plan on running Docker based bundles, you will need to install [Docker](https://docs.docker.com/) according to [the official documentation](https://docs.docker.com/installation/ubuntulinux/). Once Docker is installed then add ConductR's user/group to the `docker` group so that it has [the correct permissions in order to access Docker](http://docs.docker.com/installation/ubuntulinux/#giving-non-root-access):

```bash
[172.17.0.1]$ sudo usermod -a -G docker conductr
```

## Installing ConductR on the remaining machines

_Repeat each step in this section also on the `172.17.0.2` and `172.17.0.3` machine._

First ensure that the following ports are available between the machines forming the cluster:

* 9004 - Akka remoting for ConductR
* 9006 - Bundle streaming between ConductR nodes
* 10000 to 10999 - the default range of ports allocated to bundle component endpoints

### Installing ConductR Core on the remaining machines

The node running on the `172.17.0.1` machine is called a seed node, which is a node that is going to be used as the initial contact point when joining a cluster. Install ConductR Core on the new machine and configure the address of a seed node:

```bash
[172.17.0.2]$ sudo dpkg -i conductr_%PLAY_VERSION%_all.deb
```
or

```bash
sudo yum install conductr_%PLAY_VERSION%-1.noarch.rpm
```
then

```bash
[172.17.0.2]$ echo -DCONDUCTR_IP=$(hostname) | sudo tee -a /usr/share/conductr/conf/conductr.ini
[172.17.0.2]$ echo --seed 172.17.0.1:9004 | sudo tee -a /usr/share/conductr/conf/conductr.ini
[172.17.0.2]$ sudo service conductr restart
```

You should now see a new node in the cluster members list by using the following query:

```bash
[172.17.0.2]$ curl -s 172.17.0.2:9005/members | python3 -m json.tool
```

### Installing ConductR Agent on the remaining machines

Install ConductR Agent and configure:

```bash
[172.17.0.1]$ sudo dpkg -i conductr-agent_%PLAY_VERSION%_all.deb
```
or

```bash
[172.17.0.1]$ sudo yum install conductr-agent-%PLAY_VERSION%-1.noarch.rpm
```

then

```bash
[172.17.0.1]$ echo -Dconductr.agent.ip=$(hostname) | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ echo --core-node $(hostname):9004 | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ sudo service conductr-agent restart
```

Install optional dependencies if required. Each ConductR Agent node requires same optional dependencies to be installed.

## Installing a Proxy

_Perform each step in this section on all nodes: `172.17.0.1`, `172.17.0.2` and `172.17.0.3`. For full resilience a proxy should be installed for each machine that ConductR Agent is installed on._

Proxying application endpoints is required when running more than one instance of ConductR; which should be always for production style scenarios. Proxying endpoints permits connectivity from both external callers and for bundle components to communicate with other bundle components. This also allows an external caller to contact an application that is running on any ConductR node by contacting any proxy instance.

We will be using `HAProxy` version 1.5 or newer.

### Installing native HAProxy

Install HAProxy using the following commands.

```bash
[172.17.0.1]$ sudo apt-get -y install haproxy
```
or
```bash
[172.17.0.1]$ sudo yum install haproxy
```

On Red Hat Enterprise Linux (RHEL) 6, haproxy is in the RHEL Server Load Balancer (v6 for 64-bit x86_64) rhel-lb-for-rhel-6-server-rpms channel. You'll need to add this channel to your server.

On some Debian distributions you may need to add a dedicated Personal Package Archive (PPA) in order to install HAProxy 1.5 via the package manager. For example:

```bash
[172.17.0.1]$ sudo add-apt-repository -y ppa:vbernat/haproxy-1.5
[172.17.0.1]$ sudo apt-get update
[172.17.0.1]$ sudo apt-get -y install haproxy
```

ConductR provides a ConductR-HAProxy bundle that listens for bundle events from ConductR and updates the local HAProxy configuration file accordingly. We must specifically allow the bundle to use `sudo` to reload HAProxy.

We have the user `conductr-agent` own the HAProxy config file.

```bash
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /etc/haproxy/haproxy.cfg
```

### Installing Docker based HAProxy

We will use HAProxy version 1.5 official Docker image published to [Docker Hub](https://hub.docker.com/_/haproxy/).

#### Prerequisite

Docker needs to be installed on the host where HAProxy docker container will run.

#### Preparing HAProxy config

Setup basic HAProxy configuration for the Docker based HAProxy. Ensure that the `conductr-agent` user which runs the ConductR-HAProxy has ownership to the HAProxy config file.

```bash
[172.17.0.1]$ sudo mkdir /etc/haproxy
[172.17.0.1]$ sudo touch /etc/haproxy/haproxy.cfg
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /etc/haproxy/haproxy.cfg
```

Populate the `/etc/haproxy/haproxy.cfg` file with the following configuration.

```
defaults
    log global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend monitor
  bind :65535
  mode http
  monitor-uri /test
```

#### Running the HAProxy Docker container

Run the HAProxy Docker container.

```
[172.17.0.1]$ docker run -d --name haproxy -p 10.0.7.118:9000:9000 -p 10.0.7.118:9999:9999 -p 10.0.7.118:65535:65535 -v /etc/haproxy:/usr/local/etc/haproxy:ro haproxy:1.5
```

The container has `haproxy` as its name.

The proxy configuration located at `/etc/haproxy/haproxy.cfg`. The directory `/etc/haproxy` is mounted within the container on `/usr/local/etc/haproxy`. This will allow updates to `/etc/haproxy/haproxy.cfg` to be visible within the container.

The container exposes the port `9000`, `9999`, and `65535` to expose default port for ConductR HTTP-based endpoints, Visualizer bundle, and HAProxy test endpoint. Additional ports for your endpoints can be exposed through the `-p` option as required.

The ports exposed by the container is bound to the `10.0.7.118` address as it's the interface where external traffic is expected to arrive from. Substitute the `10.0.7.118` address with the correct address in your environment.

Run `docker ps -a` to ensure the container has been started successfully.

```
[172.17.0.1]$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                                                                               NAMES
534236e89eac        haproxy:1.5         "/docker-entrypoint.   15 seconds ago      Up 15 seconds       10.0.7.118:9000->9000/tcp, 10.0.7.118:9999->9999/tcp, 10.0.7.118:65535->65535/tcp   haproxy
```

Additional check can be performed by using `curl` command against the HAProxy test endpoint. Substitute the address `10.0.7.118` with the ip address appropriate to your environment.

```
[172.17.0.1]$ curl -v http://10.0.7.118:65535/test
*   Trying 10.0.7.118...
* Connected to 10.0.7.118 (10.0.7.118) port 65535 (#0)
> GET /test HTTP/1.1
> Host: 10.0.7.118:65535
> User-Agent: curl/7.48.0
> Accept: */*
>
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Cache-Control: no-cache
< Connection: close
< Content-Type: text/html
<
<html><body><h1>200 OK</h1>
Service ready.
</body></html>
* Closing connection 0
```

### Preparing HAProxy reload script

After updating the HAProxy configuration file, ConductR-HAProxy will signal HAProxy to reload for the updated configuration.

#### RHEL and CentOS: disable requiretty

On RHEL and CentOS it may also be neccessary to [disable default requiretty](https://bugzilla.redhat.com/show_bug.cgi?id=1020147) for the `conductr-agent` user in `sudoers`.

```bash
[172.17.0.1]$ echo 'Defaults: conductr-agent  !requiretty' | sudo tee -a /etc/sudoers
```

#### Using HAProxy reload script from the default location

HAProxy reload script is located in `/usr/bin/reloadHAProxy.sh` by default. ConductR-HAProxy will install its reload script in this location upon startup.

We will limit the bundle's sudo privileges to running `/usr/bin/reloadHAProxy.sh`. Grant permissions to the `conductr-agent` user to run the `reloadHAProxy.sh` command. An addition to `/etc/sudoers` allows for using `sudo` without password for the `reloadHAProxy.sh` script.

```bash
[172.17.0.1]$ sudo touch /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chmod 0550 /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ echo "conductr-agent ALL=(root) NOPASSWD: /usr/bin/reloadHAProxy.sh" | sudo tee -a /etc/sudoers
```

#### Using HAProxy reload script from a customized location

HAProxy reload script location can be customized. ConductR-HAProxy will install its reload script in this location upon startup.

Suppose the HAProxy reload script will be placed in `/etc/haproxy/reloadHAProxy.sh`, we will limit the bundle's sudo privileges to running this script. Grant permissions to the `conductr-agent` user to run the `reloadHAProxy.sh` command. An addition to `/etc/sudoers` allows for using `sudo` without password for the `reloadHAProxy.sh` script.

```bash
[172.17.0.1]$ sudo touch /etc/haproxy/reloadHAProxy.sh
[172.17.0.1]$ sudo chmod 0550 /etc/haproxy/reloadHAProxy.sh
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /etc/haproxy/reloadHAProxy.sh
[172.17.0.1]$ echo "conductr-agent ALL=(root) NOPASSWD: /etc/haproxy/reloadHAProxy.sh" | sudo tee -a /etc/sudoers
```

## Loading and Running ConductR-HAProxy Bundle

ConductR-HAProxy bundle listens for bundle changes within ConductR and updates the HAProxy config to expose the bundle endpoints accordingly.

### Prepare ConductR-HAProxy nodes

_Perform each step in this section on all nodes: `172.17.0.1`, `172.17.0.2` and `172.17.0.3`_.

ConductR-HAProxy bundle must be installed on all nodes where HAProxy is installed, and these nodes can be distinguished by the `haproxy` role. Assign the `haproxy` role to the nodes where the proxy will be hosted.

Append the `haproxy` role to the default `web` role as follows:

```bash
[172.17.0.1]$ echo -Dakka.cluster.roles.0=web | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ echo -Dakka.cluster.roles.1=haproxy | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ sudo service conductr-agent restart
```

### Customising Conductr-HAProxy Bundle

_These steps are necessary when using Docker-based HAProxy or using a non-default HAProxy script reload location._

These instructions for loading and running the ConductR-HAProxy bundle require the [[CLI|CLI]] to be installed. Continue with the next step only from a host where the ConductR [[CLI|CLI]] is installed.

Create a directory where the custom HAProxy configuration will be placed, e.g.

```bash
[172.17.0.1]$ mkdir -p /tmp/custom-haproxy-conf
```

Create a file called `runtime-config.sh` within the proxy configuration directory.

```bash
[172.17.0.1]$ touch /tmp/custom-haproxy-conf/runtime-config.sh
```

Populate the file with the following entry.

```bash
#!/bin/bash

CONFIG_DIR=$( cd $( dirname "${BASH_SOURCE[0]}" ) && pwd )
```

The `runtime-config.sh` configuration script will be sourced as part of the ConductR-HAProxy startup.


#### Configuration for running alongside Docker-based HAProxy

_Follow these steps when using Docker-based HAProxy._

HAProxy configuration and the reload behaviour need to be customized to support running ConductR-HAProxy Bundle alongside Docker-based HAProxy.

Create a file called `haproxy-override.cfg` within the proxy configuration directory, e.g.

```bash
[172.17.0.1]$ touch /tmp/custom-haproxy-conf/haproxy-override.cfg
```

Populate `/tmp/custom-haproxy-conf/haproxy-override.cfg` with the following.

```
defaults
    log global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend conductr-haproxy-test
  bind :65535
  mode http
  monitor-uri /test

{{#eachAcls bundles defaultHttpPort=9000}}

  {{#ifAcl 'conductr-kibana' '1' 'kibana'}}
# ConductR - Kibana Bundle HAProxy Configuration
frontend kibana_frontend
  bind {{haproxyHost}}:5601
  mode http
  acl kibana_context_root path_beg /
  use_backend kibana_backend if kibana_context_root

backend kibana_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}

  {{#ifAcl 'visualizer' '1.1' 'visualizer'}}
# ConductR - Visualizer Bundle HAProxy Configuration
frontend visualizer_frontend
  bind {{haproxyHost}}:9999
  mode http
  acl visualizer_context_root path_beg /
  use_backend visualizer_backend if visualizer_context_root

backend visualizer_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}

{{/eachAcls}}


{{#haproxyConf bundles}}
{{serviceFrontends}}
{{#unless (serviceFrontends)}}
frontend dummy
  bind 127.0.0.1:65535
{{/unless}}
{{serviceBackends}}
{{/haproxyConf}}
```

Create a file called `reloadHAProxy.sh` within the proxy configuration directory, e.g.

```bash
touch /tmp/custom-haproxy-conf/reloadHAProxy.sh
```

Populate the file with the following entry.

```bash
#!/bin/bash

set -e
docker kill -s HUP haproxy
```

Append the `/tmp/custom-haproxy-conf/runtime-config.sh` file with the following entries.

```bash
export CONDUCTR_HAPROXY_CONFIG_OVERRIDE="$CONFIG_DIR/haproxy-override.cfg"
export HAPROXY_BIND_HOST=0.0.0.0
export HAPROXY_RELOAD_SCRIPT_SOURCE="$CONFIG_DIR/reloadHAProxy.sh"
```

The `CONDUCTR_HAPROXY_CONFIG_OVERRIDE` and `HAPROXY_RELOAD_SCRIPT_SOURCE` supplies the customized HAProxy configuration and reload script respectively.

The `HAPROXY_BIND_HOST` configures the HAProxy to be bound to `0.0.0.0` address within the Docker container. This will allow the same configuration to be used regardless of the Docker IP assigned to the container. This setup should not present any additional attack surface due to the fact that access to the ports within the container must be configured explicity as part of Docker run command.

#### Configuration for customized HAProxy reload script location

_Follow this step when using a customized HAProxy reload script location._

Append the `/tmp/custom-haproxy-conf/runtime-config.sh` file with the following entries.

```bash
export HAPROXY_RELOAD_SCRIPT_LOCATION=/etc/haproxy/reloadHAProxy.sh
```

In the example above the HAProxy reload script is placed in `/etc/haproxy/reloadHAProxy.sh`, substitute this path with the path required in your environment.

#### Package the configuration bundle

Use the CLI to package the configuration override:

```bash
[172.17.0.1]$ shazar /tmp/custom-haproxy-conf
Created digested ZIP archive at ./custom-haproxy-conf-ffd0dcf76f4d565424a873022fbb39f3025d4239c87d307be3078b320988b052.zip

```

The generated file `custom-haproxy-conf-ffd0dcf76f4d565424a873022fbb39f3025d4239c87d307be3078b320988b052.zip` is the configuration override that can be loaded alongside ConductR HAProxy bundle.


### Use CLI to load and run ConductR-HAProxy bundle

_Execute the step in this section only on the `172.17.0.1` machine. We could also use a host that is not a cluster member as our control node._


If configuration bundle needs to be supplied with the Conductr-HAProxy bundle, load the ConductR-HAProxy bundle with its customized configuration and run as follows.

```bash
[172.17.0.1]$ conduct load file:/usr/share/conductr/extra/conductr-haproxy-{version}-{digest}.zip {path-to-customized-config-zip}
```

Otherwise, load the ConductR-HAProxy bundle by itself.

```bash
[172.17.0.1]$ conduct load file:/usr/share/conductr/extra/conductr-haproxy-{version}-{digest}.zip
```

Scale ConductR-HAProxy so that ConductR-HAProxy is running on every proxy node in the cluster. In our case we have 3 nodes where the proxy is expected to be running, so we scale up the ConductR-HAProxy to 3 instances.

```bash
[172.17.0.1]$ conduct run conductr-haproxy --scale 3
```

That's it! You now have a cluster of three ConductR nodes ready to start running applications. ConductR comes with a `visualizer` sample application. Head over to the next section [[CLI|CLI]] to learn how to deploy visualizer application to your fresh ConductR cluster.

## Starting and Stopping ConductR

ConductR has been designed to be "always on". If you wish to use ConductR as a transient test environment that can be stopped and started at will (for example) then you must become familiar with [how to setup for non-production](ClusterSetupConsiderations#Setting-up-for-non-production). Otherwise individual ConductR services may be stopped and started at will. Existing executions will be re-scheduled for execution elsewhere and once the new service is started, it can also receive new executions.

# EC2 Installation

> This is a tutorial for setting up a ConductR cluster on [Amazon Web Services EC2](http://aws.amazon.com/ec2/) provided in two forms. The first uses [Ansible](http://www.ansible.com) to automate the installation. The second achieves the same result using the EC2 Management Console and ssh'ing into the instances. For general installation instructions, please see [Linux Installation][#Linux-Installation).

## Prerequisites

* Amazon Web Services(AWS) account with EC2 admin rights
* Debian package of ConductR

Prior to using the Ansible playbooks to create your cluster, you will needs the following:

* Access Key and Secret values for your AWS account.
* Ansible installed on a controller host.
* An AWS Key Pair (PEM file) on the Ansible controller host.
* A copy of the ConductR deb installation package on the Ansible controller host.
* A copy of the [ConductR-Ansible](https://github.com/typesafehub/conductr-ansible) repository on the Ansible controller host.
* Ability to accept Oracle's Java License.

## Ansible Instructions

The [ConductR-Ansible](https://github.com/typesafehub/conductr-ansible) plays and playbooks provision [Lightbend ConductR](https://conductr.lightbend.com) cluster nodes in AWS EC2 using [Ansible](http://www.ansible.com).

Use create-network-ec2.yml to setup a new Virtual Private Cloud (VPC) and create your cluster in the new VPC. You only need to provide your access keys and what region to execute in. The playbook outputs a vars file for use with the build-cluster-ec.yml.

The playbook build-cluster-ec2.yml launches three instances across two availability zones. ConductR Core and ConductR Agent is installed on all instances and configured to form a cluster. The nodes are registered with a load balancer. This playbook can be used with the newly created VPC from create-network-ec2.yml or your existing VPC and security groups.

### Prepare controller host

The controller host is the host from which we will run the playbooks. The controller host should have a good, fast connection to the EC2 region we are to execute in. An Ubuntu instance running in the target EC2 region is an easy way to achieve this. See [Ansible Installation](http://docs.ansible.com/intro_installation.html) for Ansible's requirements.

From a shell on the controller host, clone the Ansible and ConductR-Ansible repositories.

```bash
sudo apt-get -y install python-setuptools autoconf g++ python2.7-dev
sudo easy_install pip
sudo pip install paramiko PyYAML Jinja2 httplib2 boto
sudo mkdir /etc/ansible
echo -e "[local]\n127.0.0.1" | sudo tee -a /etc/ansible/hosts
sudo apt-get -y install git
git clone https://github.com/ansible/ansible.git --recursive
cd ansible
source ./hacking/env-setup -q
cd ..
git clone https://github.com/typesafehub/conductr-ansible.git
cd conductr-ansible
```

Export your AWS access key id and secret.

```bash
export AWS_ACCESS_KEY_ID='ABC123'
export AWS_SECRET_ACCESS_KEY='abc123'
export ANSIBLE_HOST_KEY_CHECKING=False
```

Upload the files required for ConductR installation and your EC2 key pair file to the controller host. These files comprises of:

 * ConductR Core deb file
 * ConductR Agent deb file
 * HAProxy reload script

The ConductR installation files should be put in `conductr-ansible/conductr/files`. The file name must match the value of `CONDUCTR_PKG`, `CONDUCTR_AGENT_PKG`, and `HAPROXY_RELOAD_SCRIPT` in the vars file used.

The content of the HAProxy reload script can be found in [Installing HAProxy reload script](#Installing-HAProxy-reload-script).

Your controller host is now ready to run plays.

### Create network

ConductR-Ansible can create and prepare a new VPC for use with ConductR. Running ConductR in it's own VPC isolates the cluster from the rest of your EC2 network. If you have existing services in EC2 that ConductR needs to be able to access on the local network using an EC2 private ip address, you need to use your existing VPC. In all other cases, creating a ConductR VPC is recommended, but is not required if you are comfortabling setting up the network yourself.

```bash
ansible-playbook create-network-ec2.yml
```

Runing the playbook creates a new VPC named "ConductR VPC" in the us-east-1 region. To specify a different [EC2 region](http://docs.aws.amazon.com/general/latest/gr/rande.html#ec2_region) in which to execute, pass `EC2_REGION` using a -e key value pair. For example to execute in eu-west-1 we would use:

```bash
ansible-playbook create-network-ec2.yml -e "EC2_REGION=eu-west-1"
```

The create network playbook produces a vars file in the `vars` folder named `{{EC2_REGION}}_vars.yml` where {{EC2_REGION}} is the region used. You **must** add the name of your key pair to `{{EC2_REGION}}_vars.yml` in order to use it with the build cluster script. Change the "Key Pair Name" of `KEYPAIR: "Key Pair Name"` to that of the key pair name, which may be different than the file name and generally does not end in the .pem file extension.

If you chose to execute in a region other than us-east-1, you will also need to change the AMI value for `IMAGE` in your vars file to that of an Ubuntu image in that region. The AMI listed is the Ubuntu 14.04 LTS HVM EBS boot image published by Canonical for us-east-1. Other recent versions and types of Ubuntu instances are expected to work. The [Ubuntu AMI Locator](http://cloud-images.ubuntu.com/locator/ec2/) can help you find AMIs for alternative regions and instance types.

Re-running the create network playbook in the same EC2 region refreshes the network to the playbook configuration and does not create multiple VPCs.

### Build cluster

The second playbook launches three instances into the specified VPC and configures them into a cluster with an Elastic Load Balancer (ELB). The use of an ELB provides us with a single DNS name to access our cluster by, but its use is not required.

We pass both our vars file and EC2 PEM key file to our playbook as command line arguments. The VARS_FILE template can be the one created from the create network script. If you want to use an existing network instead, there is a `vars.yml` template you can use as a template without running the create network script.

The private-key value must be the local path and filename of the keypair that has the key pair name `KEYPAIR` specified in the vars file. For example our key pair may be named `ConductR_Key` in AWS and reside locally as `~/secrets/ConductR.pem`. In which case we would set `KEYPAIR` to `ConductR_Key` and pass `~/secrets/ConductR.pem` as our private-key argument. The private key file must be only accessible to owner and will require using `chmod 600 /path/to/{{keypair}}` if accessible to others.

```bash
ansible-playbook build-cluster-ec2.yml -e "VARS_FILE=vars/{{EC2_REGION}}_vars.yml" --private-key /path/to/{{keypair}}
```

If the playbook completes successfully, you will have a three node cluster that can be accessed using the ELB DNS name. ConductR comes with a `visualizer` sample application. The playbook created ELB includes a listener mapping port 80 to Visualizer's port 9999 port mapping.

Head over to the next section [[Managing application|ManagingApplication]] to learn how to deploy visualizer application to your fresh ConductR cluster. You can ssh into one of the cluster nodes using it's public ip address to deploy Visualizer. Use the username from the `REMOTE_USER` (currently "ubuntu") and the PEM file as for the identify file (-i). The ConductR CLI has been installed to all nodes for you. Once deployed, you can view the Visualizer via port 80 using the ELB DNS name in your browser.

Re-running this playbook launches a new set of instances. This means it can be re-run to create additional ConductR clusters. For example we might re-run the playbook to create a new cluster using a new version of ConductR to test new features. If we change only the values of `CONDUCTR_PKG`, `CONDUCTR_AGENT_PKG`, and `ELB` in the vars file to a new ConductR version package and new ELB, running the playbook again will create a new cluster using the new version in the same subnets as the previous version.

For further information about using ConductR-Ansible, please see the project [Readme](https://github.com/typesafehub/conductr-ansible/blob/master/README.md).

## Manual Instructions

This tutorial will provide you with all the key configuration details needed to run ConductR on EC2. It presumes a working knowledge of EC2 and does not provide click-by-click instructions for using EC2. Detailed instructions for all AWS steps discussed can be found in the [AWS documentation](https://aws.amazon.com/documentation/).

### Preparing EC2

Begin by preparing the EC2 network and security environment. This tutorial uses a Virtual Private Cloud (VPC) with a Classless Inter-Domain Routing (CIDR) of `10.0.0.0/16` and example addresses will be based accordingly.

#### Subnets and Security Groups
For better resilience, deploy nodes across multiple availability zones (AZ). Create three subnets in the VPC. Place each subnet in a different availability zone by specifying an AZ during subnet creation. Our example subnets names indicate their AZ. They are SN-A with a CIDR of `10.0.1.0/24`, SN-B with a CIDR of `10.0.2.0/24` and SN-C with a CIDR of `10.0.3.0/24`. Each subnet will need an internet gateway added to their route table for the destination `0.0.0.0/0`. This required so that our nodes can access the internet. All subnets can use the same route table.

Create two security groups in the VPC named SG-Nodes and SG-ELB. SG-ELB will be for our load balancer. We'll only expose port 80 and 443 to the world (`0.0.0.0/0`) here. SG-Nodes will be for the nodes. We'll need to open one or more service ports to the load balancer. When adding the inbound rule, enter the identifier for SG-ELB in the source, such as sg-a803cb4a, to allow traffic from the load balancer security group. Our service will be on port 9999 and we'll use 9009 for monitoring. We'll need to allow TCP Port 9999 and 9009 from our load balancer security group SG-ELB in to SG-Nodes, our nodes security group. Nodes will also need to communicate with each other. Add an inbound rule to allow port 9004, 9006 and port range 10000-10999 from the SG-Nodes. Finally, SG-Nodes should also allow ssh on port 22 from Anywhere (`0.0.0.0/0`) so we can also directly access our nodes from the internet.

The resultant security groups should now have the following inbound rules:

SG-ELB Inbound Rules

| Type    | Proto   | Port        | Source     |
| :------ | :-----  | :---------- | :--------- |
| HTTP    | TCP     | 80          | 0.0.0.0/0  |
| HTTPS   | TCP     | 443         | 0.0.0.0/0  |

SG-Nodes Inbound Rules

| Type	  | Proto  | Port        | Source     |
| :------ | :----- | :---------- | :--------- |
| Custom  |TCP     | 9009        | SG-ELB     |
| Custom  |TCP     | 9999        | SG-ELB     |
| Custom  |TCP     | 9004        | SG-Nodes   |
| Custom  |TCP     | 9006        | SG-Nodes   |
| Custom  |TCP     | 10000-10999 | SG-Nodes   |
| SSH     |TCP     | 22          | 0.0.0.0/0  |		

#### Load Balancer

Create an external facing load balancer from the EC2 control panel. You will need to create an internet gateway and attach it to your VPC in order to have a public load balancer. We'll add an optional HTTPS protocol listener on port 443 to the default port 80 HTTP listener. For this tutorial we will map both of our listeners to instance port 9999. Add all three subnets to the load balancer and assign the load balancer to the SG-ELB security group. Optionally you can upload an SSL Certificate to use the ELB as your TLS endpoint if you added the HTTPS listener. For health monitoring we'll use ConductR's proxy status endpoint, HTTP:9009/status. This endpoint will return an OK when ConductR's proxy has been configured.

### Preparing the AMI

Launch a single instance of the desired base AMI to use as our image master. We'll use the Ubuntu 14.04 LTS HVM EBS-SSD boot image in US-East-1, ami-76b2a71e. If you choose another base image, use an EBS boot image as they are much easy to image unless you know what your doing there. Be certain to assign a public ip address in instance details to make it easy to ssh into.

Access the console of the image instance with root access. For Ubuntu AMIs this is done as the user ubuntu using the PEM file specified at launch. The user ubuntu has sudo access. Other images will use different users. Check with the image provider for the correct user name to use.

#### Installing JRE 8

Install Java 8 as the default JRE. You will need to accept the Oracle license agreement.

```bash
sudo add-apt-repository -y ppa:webupd8team/java
sudo apt-get update
sudo apt-get -y install oracle-java8-installer && sudo apt-get clean
sudo apt-get -y install oracle-java8-set-default
echo "JAVA_HOME=/usr/lib/jvm/java-8-oracle" | sudo tee -a /etc/environment
```
#### Installing ConductR

The tutorial assumes that you have obtained the `conductr_%PLAY_VERSION%_all.deb` and `conductr-agent_%PLAY_VERSION%_all.deb` Debian package for ConductR Core and ConductR Agent respectively.

Secure copy (scp) the ConductR installation package to the image host and install ConductR Core and ConductR Agent as any other Debian package.

```bash
sudo dpkg -i conductr_%PLAY_VERSION%_all.deb
sudo dpkg -i conductr-agent_%PLAY_VERSION%_all.deb
```

ConductR Core and ConductR Agent are automatically registered as a service and started.

#### Installation miscellany

The ConductR Core service runs under the `conductr` user along with the `conductr` group. Its pid file is written to: `/var/run/conductr/running.pid` and its install location is `/usr/share/conductr`.

The ConductR Agent service runs under the `conductr-agent` user along with the `conductr-agent` group. Its pid file is written to: `/var/run/conductr-agent/running.pid` and its install location is `/usr/share/conductr-agent`.

#### Consolidated Logging

[[Consolidated logging|ConsolidatedLogging]] section describes the steps required which allow you to select the appropriate logging method for you. ConductR is bundled with an Elasticsearch based solution and is configured for that by default.

### Installing a Proxy

Proxying application endpoints is required when external communication to a service is required. We will be using `HAProxy`. Add a dedicated Personal Package Archive (PPA) and install HAProxy.

```bash
sudo add-apt-repository -y ppa:vbernat/haproxy-1.5
sudo apt-get update
sudo apt-get -y install haproxy
```

ConductR provides a ConductR-HAProxy bundle that listens for bundle events from ConductR and updates the local HAProxy configuration file accordingly. We must specifically allow the bundle to use `sudo` to reload HAProxy.

First, we have the user `conductr-agent` own the HAProxy config file.

```bash
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /etc/haproxy/haproxy.cfg
```

#### Preparing HAProxy reload script

After updating the HAProxy configuration file, ConductR-HAProxy will signal HAProxy to reload for the updated configuration.

Prepare the reload script in `/usr/bin/reloadHAProxy.sh`. ConductR-HAProxy will install its reload script in this location upon startup.

We will limit the bundle's sudo privileges to running a single script in `/usr/bin` for that purpose. Grant permissions to the `conductr-agent` user to write and run the `reloadHAPRoxy.sh` command. An addition to `/etc/sudoers` allows for using `sudo` without password for the `reloadHAProxy.sh` script. If a more specific reload sequence is required, a custom reload script can be specified using the CONDUCTR_RELOADHAPROXY_SCRIPT environment variable in a configuration bundle.

```bash
[172.17.0.1]$ sudo touch /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chmod 0770 /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ echo "conductr-agent ALL=(root) NOPASSWD: /usr/bin/reloadHAProxy.sh" | sudo tee -a /etc/sudoers
```

On RHEL and CentOS it may also be neccessary to [disable default requiretty](https://bugzilla.redhat.com/show_bug.cgi?id=1020147) for the `conductr-agent` user in `sudoers`.

```bash
[172.17.0.1]$ echo 'Defaults: conductr-agent  !requiretty' | sudo tee -a /etc/sudoers
```

#### Optional dependencies

##### Docker

ConductR supports running applications and services within Docker. If you plan on running Docker based bundles, you will need to install [Docker](https://docs.docker.com/) according to [the official documentation](https://docs.docker.com/installation/ubuntulinux/). Once Docker is installed then add ConductR's user/group to the `docker` group so that it has [the correct permissions in order to access Docker](http://docs.docker.com/installation/ubuntulinux/#giving-non-root-access):

```bash
sudo usermod -a -G docker conductr
```

### Create the AMI

With our packages installed we can create the ConductR machine image. Image the host by selecting the running instance in the EC2 dashboard and using the Create Image option from the Actions menus. We are now done with the image host and it can be terminated.

### Bring up the cluster

Once your ConductR AMI is available, launch three instances. In this tutorial we'll launch one instance into SN-A, SN-B and SN-C each so that our cluster spans three availability zones. All instances will be launched into our SG-Nodes security group. Be certain to assign public IP addresses to we can ssh into our nodes.

We will now configure ConductR on the instances and form a cluster. Repeat these steps on each of the three instances ConductR AMI.

To be able to form an inter-machine cluster, ConductR Core must be configured to listen to the machine's private host interface. This can be enabled adding a property declaration for `CONDUCTR_IP` to the start command as follows:

```bash
echo -DCONDUCTR_IP=$(hostname -i) | sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo service conductr restart
```

#### Specifying the seed node

Pick one node as the seed node and instruct the other two instances to use the other as the seed node. Here we have chosen `10.0.2.20` as the seed node and will perform this additional step on all other nodes *except* the seed node `10.0.2.20`.

```bash
echo --seed 10.0.2.20:9004 | sudo tee -a /usr/share/conductr/conf/conductr.ini
sudo service conductr restart
```

### Check the cluster
ConductR provides cluster and application information as well as its control interface via a REST API.

 ```bash
curl -s $(hostname -i):9005/members | python3 -m json.tool
```

A typical response contains the current members of the cluster (shown here is a three node cluster), the address of the node that the queried control server is running on and a list of unreachable nodes (shown here as empty).

``` json
{
    "members": [
        {
            "node": "akka.tcp://conductr@10.0.1.10:9004",
            "nodeUid": "-810451778",
            "roles": [
                "web"
            ],
            "status": "Up"
        },
        {
            "node": "akka.tcp://conductr@10.0.2.20:9004",
            "nodeUid": "280222358",
            "roles": [
                "web"
            ],
            "status": "Up"
        },
        {
            "node": "akka.tcp://conductr@10.0.3.30:9004",
            "nodeUid": "1503330106",
            "roles": [
                "web"
            ],
            "status": "Up"
        }

    ],
    "selfNode": "akka.tcp://conductr@10.0.2.20:9004",
    "selfNodeUid": "280222358",
    "unreachable": []
}

```

### Configuring ConductR Agent

_Repeat each step in this section on each node._

ConductR Agent needs to be connected to a ConductR core node in order for ConductR to run any application process. To establish this connection, configure ConductR Agent as such:

```bash
[172.17.0.1]$ echo -Dconductr.agent.ip=$(hostname) | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
[172.17.0.1]$ echo --core-node $(hostname):9004 | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
```

Once configured, restart the ConductR Agent service.

```bash
[172.17.0.1]$ sudo service conductr-agent restart
```


### Loading and Running ConductR-HAProxy Bundle

ConductR-HAProxy bundle listens for bundle changes within ConductR and updates the HAProxy config to expose the bundle endpoints accordingly.

#### Prepare ConductR-HAProxy nodes

_Repeat each step in this section on each node._

ConductR-HAProxy bundle must be installed on all nodes where HAProxy is installed, and these nodes can be distinguished by the `haproxy` role. Assign the `haproxy` role to the nodes where the proxy will be hosted.

Append the `haproxy` role to the default `web` role as follows:

```bash
echo -Dakka.cluster.roles.0=web | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
echo -Dakka.cluster.roles.1=haproxy | sudo tee -a /usr/share/conductr-agent/conf/conductr-agent.ini
sudo service conductr-agent restart
```

### Use CLI to load and run ConductR-HAProxy bundle

_Execute this step once for the entire cluster. This step would need to be repeated if the entire cluster is stopped and restarted, such as a development test lab._

These instructions for loading and running the ConductR-HAProxy bundle require the [[CLI|CLI]] to be installed. Continue with the next step once [[CLI|CLI]] is installed.

Load and run the ConductR-HAProxy bundle as follows. Scale ConductR-HAProxy so that ConductR-HAProxy is running on every proxy node in the cluster. In our case we have 3 nodes where the proxy is expected to be running, so we scale up the ConductR-HAProxy to 3 instances.

```bash
conduct load file:/usr/share/conductr/extra/conductr-haproxy-{version}-{digest}.zip
conduct run conductr-haproxy --scale 3
```

That's it! You now have a cluster of three ConductR nodes ready to start running applications.

Add all cluster instances to the load balancer. Your cluster will be reachable by the DNS name specified in the load balancer description. You can add this as a CNAME to your DNS zone file to make the cluster reachable using a hostname in your domain.

ConductR comes with a `visualizer` sample application. Head over to the next section [[CLI|CLI]] to learn how to deploy visualizer application to your fresh ConductR cluster.

# DC/OS Installation

The following guide will outline the steps to deploy and run ConductR as a framework within DC/OS.

## Prerequisite

* An existing, working DC/OS 1.7 cluster. The installation and setup of DC/OS cluster is outside of the scope of this guide.
* A working installation of Marathon in the DC/OS cluster is required to deploy ConductR into the DC/OS cluster.
* A working installation of DC/OS CLI tools successfully authenticated against the DC/OS cluster.
* A working bastion host with access to the DC/OS cluster's network is required to securely load and run ConductR bundles.


## Bastion host setup

The bastion host is a secured system of your choosing for the purpose of loading and running ConductR bundles. It is on the bastion host that we will install and run the [[ConductR CLI|CLI]]. Therefore the bastion host needs access to the ConductR Control API on port 9005 for the cluster nodes.

On AWS EC2, these requirements can be satisfied with a new custom security group in the cluster VPC. The node security groups only need accept Control API access from the bastion security group. The bastion nodes should only be accessible via SSH or similar limited, secure means.

If the bastion cannot access bundle artifacts from your own builds, you will need to copy your bundles to the bastion host in order to deploy your application bundles using the [[ConductR CLI|CLI]].


## Deploy ConductR into DC/OS cluster

Obtain the ConductR's application definition JSON.

> In order to obtain the JSON required for installations of ConductR on DC/OS then please [contact our sales department](https://www.lightbend.com/company/contact). To evaluate ConductR in general then [please visit our product page](http://www.lightbend.com/products/conductr) which provides instructions on getting started. Otherwise if you are looking to use ConductR for free from a development perspective then please [head over to our developer section](DevQuickStart).

Post the JSON to Marathon. This can be done using json mode of the 'create application' dialog. Refer to Marathon's documentation for deployment steps given the application definition JSON.

By default ConductR will be deployed with a single instance.

Wait until the ConductR instance's health to marked as healthy before proceeding.

If more instances is required, scale up to the desired number of instances, and ensure each of the ConductR instance's health to marked as healthy before proceeding. Each ConductR instance will attempt to register itself with a separate framework id, and hence it is expected to have multiple ConductRs appearing in the list of application. These ConductR instances will attempt to form a cluster of its own as part of the startup.

## Installing a Proxy

_Perform each step in this section on all public slave nodes. For full resilience a proxy should be installed for each public slave machine. The public slave machines are machines assigned with `slave_public` role._

Proxying application endpoints is required when running more than one instance of ConductR; which should be always for production style scenarios. Proxying endpoints permits connectivity from both external callers and for bundle components to communicate with other bundle components. This also allows an external caller to contact an application that is running on any ConductR node by contacting any proxy instance.

We will be using `HAProxy` version 1.5 or newer.

### Installing native HAProxy

Install HAProxy using the following commands.

```bash
[172.17.0.1]$ sudo apt-get -y install haproxy
```
or
```bash
[172.17.0.1]$ sudo yum install haproxy
```

On Red Hat Enterprise Linux (RHEL) 6, haproxy is in the RHEL Server Load Balancer (v6 for 64-bit x86_64) rhel-lb-for-rhel-6-server-rpms channel. You'll need to add this channel to your server.

On some Debian distributions you may need to add a dedicated Personal Package Archive (PPA) in order to install HAProxy 1.5 via the package manager. For example:

```bash
[172.17.0.1]$ sudo add-apt-repository -y ppa:vbernat/haproxy-1.5
[172.17.0.1]$ sudo apt-get update
[172.17.0.1]$ sudo apt-get -y install haproxy
```

ConductR provides a ConductR-HAProxy bundle that listens for bundle events from ConductR and updates the local HAProxy configuration file accordingly. We must specifically allow the bundle to use `sudo` to reload HAProxy.

We have the user `conductr-agent` own the HAProxy config file.

```bash
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /etc/haproxy/haproxy.cfg
```

Prepare the reload script in `/usr/bin/reloadHAProxy.sh`. ConductR-HAProxy will install its reload script in this location upon startup.


### Installing Docker based HAProxy

We will use HAProxy version 1.5 official Docker image published to [Docker Hub](https://hub.docker.com/_/haproxy/).

#### Prerequisite

Docker needs to be installed on the host where HAProxy docker container will run.

#### Preparing HAProxy config

Setup basic HAProxy configuration for the Docker based HAProxy. Ensure that the `conductr-agent` user which runs the ConductR-HAProxy has ownership to the HAProxy config file.

```bash
[172.17.0.1]$ sudo mkdir /etc/haproxy
[172.17.0.1]$ sudo touch /etc/haproxy/haproxy.cfg
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /etc/haproxy/haproxy.cfg
```

Populate the `/etc/haproxy/haproxy.cfg` file with the following configuration.

```
defaults
    log global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend monitor
  bind :65535
  mode http
  monitor-uri /test
```

#### Running the HAProxy Docker container

Run the HAProxy Docker container.

```
[172.17.0.1]$ docker run -d --name haproxy -p 10.0.7.118:9000:9000 -p 10.0.7.118:9999:9999 -p 10.0.7.118:65535:65535 -v /etc/haproxy:/usr/local/etc/haproxy:ro haproxy:1.5
```

The container has `haproxy` as its name.

The proxy configuration located at `/etc/haproxy/haproxy.cfg`. The directory `/etc/haproxy` is mounted within the container on `/usr/local/etc/haproxy`. This will allow updates to `/etc/haproxy/haproxy.cfg` to be visible within the container.

The container exposes the port `9000`, `9999`, and `65535` to expose default port for ConductR HTTP-based endpoints, Visualizer bundle, and HAProxy test endpoint. Additional ports for your endpoints can be exposed through the `-p` option as required.

The ports exposed by the container is bound to the `10.0.7.118` address as it's the interface where external traffic is expected to arrive from. Substitute the `10.0.7.118` address with the correct address in your environment.

Run `docker ps -a` to ensure the container has been started successfully.

```
[172.17.0.1]$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                                                                               NAMES
534236e89eac        haproxy:1.5         "/docker-entrypoint.   15 seconds ago      Up 15 seconds       10.0.7.118:9000->9000/tcp, 10.0.7.118:9999->9999/tcp, 10.0.7.118:65535->65535/tcp   haproxy
```

Additional check can be performed by using `curl` command against the HAProxy test endpoint. Substitute the address `10.0.7.118` with the ip address appropriate to your environment.

```
[172.17.0.1]$ curl -v http://10.0.7.118:65535/test
*   Trying 10.0.7.118...
* Connected to 10.0.7.118 (10.0.7.118) port 65535 (#0)
> GET /test HTTP/1.1
> Host: 10.0.7.118:65535
> User-Agent: curl/7.48.0
> Accept: */*
>
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Cache-Control: no-cache
< Connection: close
< Content-Type: text/html
<
<html><body><h1>200 OK</h1>
Service ready.
</body></html>
* Closing connection 0
```

### Preparing HAProxy reload script

After updating the HAProxy configuration file, ConductR-HAProxy will signal HAProxy to reload for the updated configuration.

#### RHEL and CentOS: disable requiretty

On RHEL and CentOS it may also be neccessary to [disable default requiretty](https://bugzilla.redhat.com/show_bug.cgi?id=1020147) for the `conductr-agent` user in `sudoers`.

```bash
[172.17.0.1]$ echo 'Defaults: conductr-agent  !requiretty' | sudo tee -a /etc/sudoers
```

#### Using HAProxy reload script from the default location

HAProxy reload script is located in `/usr/bin/reloadHAProxy.sh` by default. ConductR-HAProxy will install its reload script in this location upon startup.

We will limit the bundle's sudo privileges to running `/usr/bin/reloadHAProxy.sh`. Grant permissions to the `conductr-agent` user to run the `reloadHAProxy.sh` command. An addition to `/etc/sudoers` allows for using `sudo` without password for the `reloadHAProxy.sh` script.

```bash
[172.17.0.1]$ sudo touch /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chmod 0550 /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /usr/bin/reloadHAProxy.sh
[172.17.0.1]$ echo "conductr-agent ALL=(root) NOPASSWD: /usr/bin/reloadHAProxy.sh" | sudo tee -a /etc/sudoers
```

#### Using HAProxy reload script from a customized location

HAProxy reload script location can be customized. ConductR-HAProxy will install its reload script in this location upon startup.

Suppose the HAProxy reload script will be placed in `/etc/haproxy/reloadHAProxy.sh`, we will limit the bundle's sudo privileges to running this script. Grant permissions to the `conductr-agent` user to run the `reloadHAProxy.sh` command. An addition to `/etc/sudoers` allows for using `sudo` without password for the `reloadHAProxy.sh` script.

```bash
[172.17.0.1]$ sudo touch /etc/haproxy/reloadHAProxy.sh
[172.17.0.1]$ sudo chmod 0550 /etc/haproxy/reloadHAProxy.sh
[172.17.0.1]$ sudo chown conductr-agent:conductr-agent /etc/haproxy/reloadHAProxy.sh
[172.17.0.1]$ echo "conductr-agent ALL=(root) NOPASSWD: /etc/haproxy/reloadHAProxy.sh" | sudo tee -a /etc/sudoers
```

## Deploying bundles

[[ConductR CLI|CLI]] requires the host or ip address of one of the ConductR node to deploy and run bundles.

Obtain the host or ip address of one of the ConductR Core nodes that's currently running in the DCOS cluster using the `dcos task` command.

```bash
$ dcos task
NAME                    HOST       USER  STATE  ID
ConductR                10.0.3.76  root    R    conductr.6e9251af-4e28-11e6-8edd-4219f69bc956
```

In the example above ConductR has been deployed to `10.0.3.76` - this is the ip address to be obtained. If ConductR has been deployed to multiple nodes, the host address from any these nodes can be selected.

SSH to the bastion host. Once logged in, export the `CONDUCTR_IP` environment variable with the obtained address.

You are now ready to [[deploy bundles|DeployingBundlesOps]] into ConductR.


## Loading and Running ConductR-HAProxy Bundle

ConductR-HAProxy bundle will expose services provided by the bundle so it can be accessed from outside cluster. ConductR-HAProxy will ensure the HAProxy configuration is kept update based on the bundles which are running.

### Obtaining ConductR-HAProxy Bundle

First we need to obtain the `.tgz` ConductR installations package.

> In order to obtain the installations of ConductR then please [contact our sales department](https://www.lightbend.com/company/contact). To evaluate ConductR in general then [please visit our product page](http://www.lightbend.com/products/conductr) which provides instructions on getting started. Otherwise if you are looking to use ConductR for free from a development perspective then please [head over to our developer section](DevQuickStart).

Once the ConductR `.tgz` is obtained, extract ConductR HAProxy bundle from the package. The ConductR HAProxy bundle can be found within the ConductR package under  `conductr-{version}/extra/conductr-haproxy-{version}-{digest}.zip`.

Upload the ConductR HAProxy bundle to the bastion host, e.g. the `/tmp` directory.

SSH to the bastion host. Once logged in, export the `CONDUCTR_IP` environment variable with the host or ip address of one of the ConductR Core that's currently running.

### Customising Conductr-HAProxy Bundle

_These steps are necessary when using Docker-based HAProxy or using a non-default HAProxy script reload location._

Create a directory where the custom HAProxy configuration will be placed, e.g.

```bash
[172.17.0.1]$ mkdir -p /tmp/custom-haproxy-conf
```

Create a file called `runtime-config.sh` within the proxy configuration directory.

```bash
[172.17.0.1]$ touch /tmp/custom-haproxy-conf/runtime-config.sh
```

Populate the file with the following entry.

```bash
#!/bin/bash

CONFIG_DIR=$( cd $( dirname "${BASH_SOURCE[0]}" ) && pwd )
```

The `runtime-config.sh` configuration script will be sourced as part of the ConductR-HAProxy startup.


#### Configuration for running alongside Docker-based HAProxy

_Follow these steps when using Docker-based HAProxy._

HAProxy configuration and the reload behaviour need to be customized to support running ConductR-HAProxy Bundle alongside Docker-based HAProxy.

Create a file called `haproxy-override.cfg` within the proxy configuration directory, e.g.

```bash
[172.17.0.1]$ touch /tmp/custom-haproxy-conf/haproxy-override.cfg
```

Populate `/tmp/custom-haproxy-conf/haproxy-override.cfg` with the following.

```
defaults
    log global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend conductr-haproxy-test
  bind :65535
  mode http
  monitor-uri /test

{{#eachAcls bundles defaultHttpPort=9000}}

  {{#ifAcl 'conductr-kibana' '1' 'kibana'}}
# ConductR - Kibana Bundle HAProxy Configuration
frontend kibana_frontend
  bind {{haproxyHost}}:5601
  mode http
  acl kibana_context_root path_beg /
  use_backend kibana_backend if kibana_context_root

backend kibana_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}

  {{#ifAcl 'visualizer' '1.1' 'visualizer'}}
# ConductR - Visualizer Bundle HAProxy Configuration
frontend visualizer_frontend
  bind {{haproxyHost}}:9999
  mode http
  acl visualizer_context_root path_beg /
  use_backend visualizer_backend if visualizer_context_root

backend visualizer_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}

{{/eachAcls}}


{{#haproxyConf bundles}}
{{serviceFrontends}}
{{#unless (serviceFrontends)}}
frontend dummy
  bind 127.0.0.1:65535
{{/unless}}
{{serviceBackends}}
{{/haproxyConf}}
```

Create a file called `reloadHAProxy.sh` within the proxy configuration directory, e.g.

```bash
touch /tmp/custom-haproxy-conf/reloadHAProxy.sh
```

Populate the file with the following entry.

```bash
#!/bin/bash

set -e
docker kill -s HUP haproxy
```

Append the `/tmp/custom-haproxy-conf/runtime-config.sh` file with the following entries.

```bash
export CONDUCTR_HAPROXY_CONFIG_OVERRIDE="$CONFIG_DIR/haproxy-override.cfg"
export HAPROXY_BIND_HOST=0.0.0.0
export HAPROXY_RELOAD_SCRIPT_SOURCE="$CONFIG_DIR/reloadHAProxy.sh"
```

The `CONDUCTR_HAPROXY_CONFIG_OVERRIDE` and `HAPROXY_RELOAD_SCRIPT_SOURCE` supplies the customized HAProxy configuration and reload script respectively.

The `HAPROXY_BIND_HOST` configures the HAProxy to be bound to `0.0.0.0` address within the Docker container. This will allow the same configuration to be used regardless of the Docker IP assigned to the container. This setup should not present any additional attack surface due to the fact that access to the ports within the container must be configured explicity as part of Docker run command.

#### Configuration for customized HAProxy reload script location

_Follow this step when using a customized HAProxy reload script location._

Append the `/tmp/custom-haproxy-conf/runtime-config.sh` file with the following entries.

```bash
export HAPROXY_RELOAD_SCRIPT_LOCATION=/etc/haproxy/reloadHAProxy.sh
```

In the example above the HAProxy reload script is placed in `/etc/haproxy/reloadHAProxy.sh`, substitute this path with the path required in your environment.

#### Package the configuration bundle

Use the CLI to package the configuration override:

```bash
[172.17.0.1]$ shazar /tmp/custom-haproxy-conf
Created digested ZIP archive at ./custom-haproxy-conf-ffd0dcf76f4d565424a873022fbb39f3025d4239c87d307be3078b320988b052.zip

```

The generated file `custom-haproxy-conf-ffd0dcf76f4d565424a873022fbb39f3025d4239c87d307be3078b320988b052.zip` is the configuration override that can be loaded alongside ConductR HAProxy bundle.


### Use CLI to load and run ConductR-HAProxy bundle

If configuration bundle needs to be supplied with the Conductr-HAProxy bundle, load the ConductR-HAProxy bundle with its customized configuration and run as follows.

```bash
[172.17.0.1]$ conduct load file:/tmp/conductr-haproxy-{version}-{digest}.zip {path-to-customized-config-zip}
```

Otherwise, load the ConductR-HAProxy bundle by itself.

```bash
[172.17.0.1]$ conduct load file:/tmp/conductr-haproxy-{version}-{digest}.zip
```

Scale ConductR-HAProxy so that ConductR-HAProxy is running on every proxy node in the cluster. In our case we have 3 nodes where the proxy is expected to be running, so we scale up the ConductR-HAProxy to 3 instances.

```bash
[172.17.0.1]$ conduct run conductr-haproxy --scale 3
```

## Verifying if bundle has been started successfully

Use the command `conduct info` from the bastion host to verify successful startup of the bundle.

```bash
$ conduct info
ID       NAME        #REP  #STR  #RUN
6e68f05  visualizer     1     0     1
```

In the example above, a bundle called `visualizer` has been started successfully.

This can also be confirmed by executing the `dcos task` command which will display the bundle running as a task from the context of DC/OS. The `visualizer-1` task belongs to the currently running `visualizer` bundle having `1` as the `compatibilityVersion`.

```bash
$ dcos task
NAME           HOST       USER  STATE  ID
visualizer-1   10.0.3.75  root    R    6e68f055d1f5715ad3ff19172fa5efaf_0f6e119c-9288-425a-89e3-36379dcaccda
```

The `compatibilityVersion` is explained in the to [bundle configuration](BundleConfiguration).
