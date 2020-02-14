---
title: "OpenShift 4 Lab: Libvirt Baremetal UPI"
date: 2020-02-14T00:30:41+08:00
author: Muhammad Aizuddin
categories:
  - OpenShift
tags:
  - OpenShift
  - Libvirt
thumbnail: https://upload.wikimedia.org/wikipedia/commons/3/3a/OpenShift-LogoType.svg
---

# ___Introduction___

Hello everyone, just a couple of month ago, Red Hat has released a shiny [OpenShift 4](https://docs.openshift.com "OpenShift 4 Docs") based on CoreOS technology. In this guide, we going to see how we can install OCP4 UPI on libvirt.

### What is UPI and IPI?

**UPI**: User Provided Infrastructure (User manually prepare machine for bootstrap)  
**IPI**: Installer Provided Infrastructure (OCP Installer automatically terraforming machine for bootstrap)


### PreReq

1. A Valid Red Hat OpenShift subscription
2. DNS Server (Helper Node)
3. HAProxy Server (Helper Node)
4. DHCP Server (Helper Node)
5. PXE/TFTP Server (Helper Node)
6. Libvirt Host
7. VM Requirement:

| Node        | vCPU | Memory(GB) | Disk(GB) | OS      | Hostname            | Count |
|-------------|------|------------|----------|---------|---------------------|-------| 
| Helper(Bastion)      |   4  |      4     |    50    | RHEL 8  | helper.cldshft.com |   1   |
| Bootstrap | 4 | 6 | 50 | RHCOS |bootstrap.cldshft.com | 1| 
| Master | 4 |8 | 50 | RHCOS | master[01-03].cldshft.com | 3 |
| Worker | 4 | 8 | 50 | RHCOS | worker[01-03].cldshft.com | 3 |   

**NOTE**: Ensure boot order are correct, should only boot __ONCE__ using network.

For OpenShift SDN Requirement:

| CIDR | Value |
|------------------|--------------|
| Cluster Network | 10.128.0.0/14 |
| Cluster Network Host Prefix | /23 |
| Service Network | 172.30.0.0/16 |

# ___Steps___

In brief, what we going to do:  
  * Configure WebServer, DNS, DHCP and PXE on Helper Node.  
  * Configure HAProxy on Helper Node.  
  * Download and prepare OpenShift installer and client binaries on Helper Node.  
  * Prepare Ignition files.  
  * Bootup all OCP VMs.

### A. Configure DNS  

1. Install bind packages  

        root@helper #> dnf -y install bind bind-utils

2. Comment out below lines from **/etc/named.conf**:
   
        #listen-on port 53 { 127.0.0.1; };  
        #listen-on-v6 port 53 { ::1; };

3. Allow query from the subnet, set this in **/etc/named.conf**:
   
        allow-query     { localhost;192.168.0.0/24; };

4. Create a Forward DNS zone in **/etc/named.conf**:

        zone "cldshft.com" IN {
            type master;
            file "cldshft.com.db"; 
            allow-update { none; };
        };


5. Create a Reverse DNS zone in **/etc/named.conf**:

        zone “50.168.192.in-addr.arpa” {
            type master;
            file “/var/named/50.168.192.in-addr.arpa”
        };

6. Create a zone database file in /var/named/cldshft.com.db with below content:

        $TTL      60
        @         IN  SOA dns.ocp4.cldshft.com. root.cldshft.com. (
                       2019022400 ; serial
                       3h         ; refresh
                       15         ; retry
                       1w         ; expire
                       3h         ; minimum
                                                                        )
                  IN  NS  dns.ocp4.cldshft.com.
        dns.ocp4                        IN  A       192.168.50.30
        bootstrap.ocp4                  IN  A       192.168.50.60
        master01.ocp4                   IN  A       192.168.50.61
        master02.ocp4                   IN  A       192.168.50.62
        master03.ocp4                   IN  A       192.168.50.63
        etcd-0.ocp4                     IN  CNAME   master01.ocp4
        etcd-1.ocp4                     IN  CNAME   master02.ocp4
        etcd-2.ocp4                     IN  CNAME   master03.ocp4
        api.ocp4                        IN  A       192.168.50.30
        api-int.ocp4                    IN  A       192.168.50.30
        *.apps.ocp4                     IN  A       192.168.50.30
        worker01.ocp4                   IN  A       192.168.50.64
        worker02.ocp4                   IN  A       192.168.50.65
        worker03.ocp4                   IN  A       192.168.50.66
        _etcd-server-ssl._tcp.ocp4      IN  SRV     0 10    2380 etcd-0.ocp4
        _etcd-server-ssl._tcp.ocp4      IN  SRV     0 10    2380 etcd-1.ocp4
        _etcd-server-ssl._tcp.ocp4      IN  SRV     0 10    2380 etcd-2.ocp4 

7. Create a zone database file in /var/named/50.168.192.in-addr.arpa with below content:  
   
        $TTL    60 
        @       IN SOA localhost. root.localhost. (
                        991079290 ; serial
                        28800 ; refresh
                        14400 ; retry
                        3600000 ; expire
                        86400 ; default_ttl
                                                )
                IN NS dns.ocp4.cldshft.com.
        30     IN      PTR     dns.ocp4.cldshft.com
        60     IN      PTR     bootstrap.ocp4.cldshft.com
        61     IN      PTR     master01.ocp4.cldshft.com
        62     IN      PTR     master02.ocp4.cldshft.com
        63     IN      PTR     master03.ocp4.cldshft.com
        64     IN      PTR     worker01.ocp4.cldshft.com
        65     IN      PTR     worker02.ocp4.cldshft.com
        66     IN      PTR     worker03.ocp4.cldshft.com

8. Enable and start named service.
        
        root@helper #> firewall-cmd --add-service=dns --permanent
        root@helper #> firewall-cmd --reload
        root@helper #> systemctl start named --now

### B. Configure DHCP/PXE

We going to do PXE Booting and let boot image to scan TFTP server and  <a href="https://docs.oracle.com/cd/E24628_01/em.121/e27046/appdx_pxeboot.htm#EMLCM1219" target="_blank">pull corresponding boot configuration files</a> as per VM MAC Address.


| VM Name | MAC Address | PXE Boot Configuration Filename|
|------------------|--------------|----------|
| bootstrap | 52:54:00:7d:2d:b1 | 52-54-00-7d-2d-b1 |
| master01  | 52:54:00:7d:2d:b2 | 52-54-00-7d-2d-b2 |
| master02 | 52:54:00:7d:2d:b3 | 52-54-00-7d-2d-b3 |
| master03| 52:54:00:7d:2d:b4 | 52-54-00-7d-2d-b4 |
| worker01 | 52:54:00:7d:2d:b5 | 52-54-00-7d-2d-b5 |
| worker02 | 52:54:00:7d:2d:b6 | 52-54-00-7d-2d-b6 |
| worker03 | 52:54:00:7d:2d:c1 | 52-54-00-7d-2d-c1 |

1. Install bind packages  

        root@helper #> dnf -y install dhcp-server tftp-server syslinux


2. Configure **/etc/dhcp/dhcpd.conf**:
   
        ddns-update-style interim;
        ignore client-updates;
        authoritative;
        allow booting;
        allow bootp;
        deny unknown-clients;
        subnet 192.168.50.0 netmask 255.255.255.0 {
                option routers 192.168.50.1;
                option domain-name-servers 192.168.50.30;
                option ntp-servers time.unisza.edu.my;
                option domain-search "cldshft","ocp4.cldshft.com";
                filename "pxelinux.0";
                next-server 192.168.50.30;

                host bootstrap { hardware ethernet 52:54:00:7d:2d:b1; fixed-address 192.168.50.60; option host-name "bootstrap"; }
                host master01 { hardware ethernet 52:54:00:7d:2d:b2; fixed-address 192.168.50.61; option host-name "master01"; }
                host master02 { hardware ethernet 52:54:00:7d:2d:b3; fixed-address 192.168.50.62; option host-name "master02"; }
                host master03 { hardware ethernet 52:54:00:7d:2d:b4; fixed-address 192.168.50.63; option host-name "master03"; }
                host worker01 { hardware ethernet 52:54:00:7d:2d:b5; fixed-address 192.168.50.64; option host-name "worker01"; }
                host worker02 { hardware ethernet 52:54:00:7d:2d:b6; fixed-address 192.168.50.65; option host-name "worker02"; }
                host worker03 { hardware ethernet 52:54:00:7d:2d:c1; fixed-address 192.168.50.66; option host-name "worker03"; }
        }
 
     **NOTE:** This dhcpd server configuration **doest not offered any IP range lease** (range option absent) and **deny any client** (*deny unknown-clients* option) that is not part of the host declarations. I.e static dhcp configuration.


3. Now, referring back to *MAC Address* table above, we goint to create three type of boot config, bootstrap, master and worker.

4. For bootstrap node, create file  **/var/lib/tftpboot/pxelinux.cfg/52-54-00-7d-2d-b1** with below content:
   
        default menu.c32
        prompt 0
        timeout 2
        menu title **** OpenShift 4 Bootstrap PXE Boot Menu ****
 
        label Install CoreOS 4.3.0 Bootstrap Node
          kernel /openshift4/4.3.0/rhcos-4.3.0-x86_64-installer-kernel
          append ip=dhcp rd.neednet=1 coreos.inst.install_dev=vda console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.image_url=http://192.168.50.30:8080/openshift4/images/rhcos-4.3.0-x86_64-metal.raw.gz coreos.inst.ignition_url=http://192.168.50.30:8080/openshift4/4.3.0/ignitions/bootstrap.ign initrd=/openshift4/4.3.0/rhcos-4.3.0-x86_64-installer-initramfs.img

  
5. For first master node. create file **/var/lib/tftpboot/pxelinux.cfg/52-54-00-7d-2d-b2** with below content (**do this for all masters, remember to use correct PXE Boot Configuration filename**):  
   
        default menu.c32
        prompt 0
        timeout 2
        menu title **** OpenShift 4 Masters PXE Boot Menu ****
 
        label Install CoreOS 4.3.0 Master Node
          kernel /openshift4/4.3.0/rhcos-4.3.0-x86_64-installer-kernel
          append ip=dhcp rd.neednet=1 coreos.inst.install_dev=vda console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.image_url=http://192.168.50.30:8080/openshift4/images/rhcos-4.3.0-x86_64-metal.raw.gz coreos.inst.ignition_url=http://192.168.50.30:8080/openshift4/4.3.0/ignitions/master.ign initrd=/openshift4/4.3.0/rhcos-4.3.0-x86_64-installer-initramfs.img


6. For first worker node. create file **/var/lib/tftpboot/pxelinux.cfg/52-54-00-7d-2d-b5** with below content (**do this for all workers, remember to use correct PXE Boot Configuration filename**):  

        default menu.c32
        prompt 0
        timeout 2
        menu title **** OpenShift 4 Workers PXE Boot Menu ****
 
        label Install CoreOS 4.3.0 Worker Node
          kernel /openshift4/4.3.0/rhcos-4.3.0-x86_64-installer-kernel
          append ip=dhcp rd.neednet=1 coreos.inst.install_dev=vda console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.image_url=http://192.168.50.30:8080/openshift4/images/rhcos-4.3.0-x86_64-metal.raw.gz coreos.inst.ignition_url=http://192.168.50.30:8080/openshift4/4.3.0/ignitions/worker.ign initrd=/openshift4/4.3.0/rhcos-4.3.0-x86_64-installer-initramfs.img

8. Now, we goint to host all necessary for PXE boot i.e kernel and initramfs, create a folder and download from Red Hat CDN:
        
        root@helper #> mkdir -p /var/lib/tftpboot/openshift4/4.3.0
        root@helper #> cd /var/lib/tftpboot/openshift4/4.3.0
        root@helper #> wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/4.3.0/rhcos-4.3.0-x86_64-installer-kernel
        root@helper #> wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/4.3.0/rhcos-4.3.0-x86_64-installer-initramfs.img
        root@helper #> restorecon -RFv .


9.  Copy *syslinux* required file to **/var/lib/tftpboot** and start dhcpd and tftp server:
   
        root@helper #> cp -rvf /usr/share/syslinux/* /var/lib/tftpboot
        root@helper #> firewall-cmd --add-service=tftp --add-service=tftp-client --add-service=dhcp
        root@helper #> firewall-cmd --reload
        root@helper #> systemctl enable tftp --now
        root@helper #> systemctl enable dhcpd --now


### C. Configure WebServer

We are going to use this WebServer to host RHCOS image for kickstart process.

1. Install httpd and allow firewall:
   
        root@helper #> yum install httpd -y
        root@helper #> firewall-cmd --add-port=8080 --permanent
        root@helper #> firewall-cmd --reload

2. Change http listen port directive to listen on port 8080 (Since we going to use this node as HAProxy that will listen on port  80 later causing conflict since we are on the same IP address). Change **/etc/httpd/httpd.conf** *Listen 80* to:
   
        Listen 8080

3. Now, we goint to host all necessary file to install RHCOS, ie. OS image:
   
        root@helper #> mkdir -p /var/www/html/openshift4/4.3.0/images/
        root@helper #> cd /var/www/html/openshift4/4.3.0/images/
        root@helper #> wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/4.3.0/rhcos-4.3.0-x86_64-metal.raw.gz
        root@helper #> restorecon -RFv .
        root@helper #> chown -Rvf apache. /var/www/html/

4. Enable and start HTTPD server:
   
        root@helper #> systemctl enable httpd --now


### D. Configure HAProxy as Network Load Balancer

1. Install haproxy, allow firewall, allow SElinux:
   
        root@helper #> yum install haproxy -y
        root@helper #> firewall-cmd --add-port={6443/tcp,22623/tcp,8080/tcp,32700/tcp} --permanent
        root@helper #> firewall-cmd --reload

        root@helper #>  semanage port  -a 22623 -t http_port_t -p tcp
        root@helper #>  semanage port  -a 6443 -t http_port_t -p tcp
        root@helper #>  semanage port  -a 32700 -t http_port_t -p tcp

        root@helper #>  semanage port  -l  | grep -w http_port_t
        http_port_t                    tcp      22623, 32700, 6443, 80, 81, 443, 488, 8008, 8009, 8443, 9000


2. Configure **/etc/haproxy/haproxy.conf** with below content:

        ---------------------------------------------------------------------
        round robin balancing for OCP Kubernetes API Server
        ---------------------------------------------------------------------
        frontend k8s_api
                bind *:6443
                mode tcp
                default_backend k8s_api_backend
        
        backend k8s_api_backend
                balance roundrobin
                mode tcp
                server bootstrap 192.168.50.60:6443 check
                server master01 192.168.50.61:6443 check
                server master02 192.168.50.62:6443 check
                server master03 192.168.50.63:6443 check
        
        ---------------------------------------------------------------------
        round robin balancing for OCP Machine Config Server
        ---------------------------------------------------------------------
        frontend machine_config
                bind *:22623
                mode tcp
                default_backend machine_config_backend
        backend machine_config_backend
                balance roundrobin
                mode tcp
                server bootstrap 192.168.50.60:22623 check
                server master01 192.168.50.61:22623 check
                server master02 192.168.50.62:22623 check
                server master03 192.168.50.63:22623 check
        
        ---------------------------------------------------------------------
        round robin balancing for OCP Ingress Insecure Port
        ---------------------------------------------------------------------
        frontend ingress_insecure
                bind *:80
                mode tcp
                default_backend ingress_insecure_backend
        backend ingress_insecure_backend
                balance roundrobin
                mode tcp
                server worker01 192.168.50.64:80 check
                server worker02 192.168.50.65:80 check
                server worker03 192.168.50.66:80 check
        
        ---------------------------------------------------------------------
        round robin balancing for OCP Ingress Secure Port
        ---------------------------------------------------------------------
        frontend ingress_secure
                bind *:443
                mode tcp
                default_backend ingress_secure_backend
        backend ingress_secure_backend
                balance roundrobin
                mode tcp
                server worker01 192.168.50.64:443 check
                server worker02 192.168.50.65:443 check
                server worker03 192.168.50.66:443 check
        
        ---------------------------------------------------------------------
        Exposing HAProxy Statistic Page
        ---------------------------------------------------------------------
        listen stats
                bind :32700
                stats enable
                stats uri /
                stats hide-version
                stats auth admin:password

3. Enable and start HAProxy server:
   
        root@helper #> systemctl enable haproxy --now


4. Validate the haproxy by accessing console at *http://192.168.50.30:32700* with password declared in the config file.



### E. Configure OpenShift Installer and required binaries

In this step we going to download OCP installer and client binaries from Red Hat CDN.

1. Download and extract installer and client binary:
   
        root@helper #> https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux-4.3.1.tar.gz
        root@helper #> https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux-4.3.1.tar.gz
        root@helper #> tar -xvf openshift-install-linux-4.3.1.tar.gz -C /usr/bin/
        root@helper #> tar -xvf openshift-client-linux-4.3.1.tar.gz-C /usr/bin/

2. Create installation folder, **~/ocp4** and install-config-base.yaml:
   
        root@helper #> mkdir -p ~/ocp4
        root@helper #> cat install-config-base.yaml 
        apiVersion: v1
        baseDomain: techbeatly.com
        compute:
        - hyperthreading: Enabled
          name: worker
          replicas: 0
        controlPlane:
          hyperthreading: Enabled
          name: master
          replicas: 3
        metadata:
          name: ocp4
        networking:
          clusterNetworks:
          - cidr: 10.128.0.0/14
            hostPrefix: 23
          networkType: OpenShiftSDN
          serviceNetwork:
          - 172.30.0.0/16
        platform:
          none: {}
        pullSecret: 'Paste PullSecret from cloud.redhat.com'
        sshKey: 'Paste SSH public key'

    **NOTE:** We create install-config-base.yaml due to install-config.yaml will be consumed by the installer and get deleted after manifest for ignition file created.

3. Copy **install-config-base.yaml** to **install-config.yaml** and execute installer binary to create ignition file:
   
        root@helper #> cp install-config-base.yaml install-config.yaml 

        root@helper #>  openshift-install create ignition-configs  
        WARNING There are no compute nodes specified. The cluster will not fully initialize without compute nodes. 
        INFO Consuming "Install Config" from target directory     

4. This will create ignition file and associated files (**DO NOT REUSE IGNITION FILE AFTER 24 HOURS!**):
   
         root@helper #> ll
         total 288
         drwxr-xr-x. 2 root root     50 Jul  1 16:46 auth
        -rw-r--r--. 1 root root 276768 Jul  1 16:46 bootstrap.ign
        -rw-r--r--. 1 root root   3545 Jul  1 16:42 install-config-base.yaml
        -rw-r--r--. 1 root root   1824 Jul  1 16:46 master.ign
        -rw-r--r--. 1 root root     96 Jul  1 16:46 metadata.json
        -rw-r--r--. 1 root root   1824 Jul  1 16:46 worker.ign

5. Now we going to copy *.ign file to WebServer hosting directory for RHCOS to pull during kickstart:
   
        root@helper #> mkdir /var/www/html/openshift4/4.3.0/ignitions
        root@helper #> cp -v *.ign /var/www/html/openshift4/4.3.0/ignitions/
        'bootstrap.ign' -> '/var/www/html/openshift4/4.3.0/ignitions/bootstrap.ign'
        'master.ign' -> '/var/www/html/openshift4/4.3.0/ignitions/master.ign'
        'worker.ign' -> '/var/www/html/openshift4/4.3.0/ignitions/worker.ign'
 
        root@helper #> restorecon -RFv /var/www/html/
        Relabeled /var/www/html/openshift4 from unconfined_u:object_r:httpd_sys_content_t:s0 to system_u:object_r:httpd_sys_content_t:s0
        Relabeled /var/www/html/openshift4/4.3.0 from unconfined_u:object_r:httpd_sys_content_t:s0 to system_u:object_r:httpd_sys_content_t:s0
        Relabeled /var/www/html/openshift4/4.3.0/ignitions from unconfined_u:object_r:httpd_sys_content_t:s0 to system_u:object_r:httpd_sys_content_t:s0
        Relabeled /var/www/html/openshift4/4.3.0/ignitions/bootstrap.ign from unconfined_u:object_r:httpd_sys_content_t:s0 to system_u:object_r:httpd_sys_content_t:s0
        Relabeled /var/www/html/openshift4/4.3.0/ignitions/master.ign from unconfined_u:object_r:httpd_sys_content_t:s0 to system_u:object_r:httpd_sys_content_t:s0
        Relabeled /var/www/html/openshift4/4.3.0/ignitions/worker.ign from unconfined_u:object_r:httpd_sys_content_t:s0 to system_u:object_r:httpd_sys_content_t:s0

        root@helper #> chown -Rvf apache. /var/www/html/

### E. Start all VM and kickstart!

At this stage, start all VMs and let it automatically boot into pre-configured boot config with corresponding ignition file. No human interaction required.


<img src="https://raw.githubusercontent.com/technoshift/cloudshift/master/img/masterpxe.png" alt="Master Boot">

And from helper node log, e.g for master01:

        Feb 14 07:12:01 helper dhcpd[1747]: DHCPDISCOVER from 52:54:00:7d:2d:b2 via eth0
        Feb 14 07:12:01 helper dhcpd[1747]: DHCPOFFER on 192.168.50.61 to 52:54:00:7d:2d:b2 via eth0
        Feb 14 07:12:01 helper dhcpd[1747]: DHCPREQUEST for 192.168.50.61 (192.168.50.30) from 52:54:00:7d:2d:b2 via eth0
        Feb 14 07:12:01 helper dhcpd[1747]: DHCPACK on 192.168.50.61 to 52:54:00:7d:2d:b2via eth0
        Feb 14 07:12:10 helper dhcpd[1747]: DHCPDISCOVER from 52:54:00:7d:2d:b2 via eth0
        Feb 14 07:12:10 helper dhcpd[1747]: DHCPOFFER on 192.168.50.61 to 52:54:00:7d:2d:b2 via eth0
        Feb 14 07:12:14 helper dhcpd[1747]: DHCPDISCOVER from 52:54:00:7d:2d:b2 via eth0
        Feb 14 07:12:14 helper dhcpd[1747]: DHCPOFFER on 192.168.50.61 to 52:54:00:7d:2d:b2 via eth0
        Feb 14 07:12:22 helper dhcpd[1747]: DHCPREQUEST for 192.168.50.61 (192.168.50.30) from 52:54:00:7d:2d:b2 via eth0
        Feb 14 07:12:22 helper dhcpd[1747]: DHCPACK on 192.168.50.61 to 52:54:00:7d:2d:b2 via eth0
        Feb 14 07:12:26 helper in.tftpd[2494]: Client ::ffff:192.168.50.61 finished pxelinux.0
        Feb 14 07:12:26 helper in.tftpd[2495]: Client ::ffff:192.168.50.61 finished ldlinux.c32
        Feb 14 07:12:26 helper in.tftpd[2496]: Client ::ffff:192.168.50.61 finished pxelinux.cfg/52-54-00-7d-2d-b2


Installation will go on from this point, in 10,000 feet view, this how the process looks like:

  * Bootstrap stated temporary API and Machine Config Server.
  * Master started etcd and wait for all peers to come up.
  * Bootstrap will inject initial proposal to Masters etcd.
  * Bootstrap approve all node CSRs.
  * Bootstrap and master continue to start all necessary control plane and services.
  * Bootstrap stopped it services.
  * It is safe now to delete bootstrap node.

 **NOTE:** Installation time taken are proportinal to infrastructure performance. If you have low network bandwidth, low performance infra, patient is the key. It should not take more than one hour to get it running for typical infrastructure performance.