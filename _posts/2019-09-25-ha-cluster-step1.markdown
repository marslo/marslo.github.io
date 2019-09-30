---
layout: post
title:  1.15.3 HA kubernetes Cluster Playbook (basic environment)
date:   2019-09-25 16:47:24 +0800
categories: ["kubernetes"]
---

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Extended etcd Topology](#extended-etcd-topology)
  - [Kubernetes HA cluster with external etcd](#kubernetes-ha-cluster-with-external-etcd)
  - [Server Matrix](#server-matrix)
    - [Environment List](#environment-list)
    - [`/etc/hosts`](#etchosts)
    - [variables](#variables)
- [Tools Setup](#tools-setup)
  - [cfssl & cfssljson](#cfssl--cfssljson)
  - [etcd](#etcd)
  - [keepalived](#keepalived)
  - [haproxy 2.0.6](#haproxy-206)
  - [helm](#helm)
  - [docker](#docker)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Extended etcd Topology
## [Kubernetes HA cluster with external etcd](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#external-etcd-topology)

<img src="{{site.url}}/images/external-etcd-topology.png" style="width: 666px;" />

## Server Matrix

### Environment List

```bash
| Hostname   | IP Address      | etcd              |
|------------|-----------------|-------------------|
| master01   | 192.168.100.200 | master01 (etcd-0) |
| master02   | 192.168.100.201 | master02 (etcd-1) |
| master03   | 192.168.100.202 | master03 (etcd-2) |
| node01     | 192.168.100.203 |                   |
| node02     | 192.168.100.204 |                   |
|            |                 |                   |
| virtual IP | 192.168.100.250 |                   |
```

### `/etc/hosts`

Add for all servers

```bash
192.168.100.200     master01
192.168.100.201     master02
192.168.100.202     master03
192.168.100.203     worker01
192.168.100.204     worker02

192.168.100.250     jenkins.marslo.com
192.168.100.250     prometheus.marslo.com
192.168.100.250     grafana.marslo.com
192.168.100.250     dashboard.marslo.com
```

### variables
```bash
# hostname
master01Name='master01'
master02Name='master02'
master03Name='master03'

# ipaddress
master01IP='192.168.100.200'
master01IP='192.168.100.201'
master01IP='192.168.100.202'
virtualIP='192.168.100.250'

leadIP="${master01IP}"
leadName="${master01Name}"

k8sVer='v1.15.3'
cfsslDownloadUrl='https://pkg.cfssl.org/R1.2'

etcdVer='v3.3.15'
etcdDownloadUrl='https://github.com/etcd-io/etcd/releases/download'
etcdSSLPath='/etc/etcd/ssl'
etcdInitialCluster="${master01Name}=https://${master01IP}:2380,${master02Name}=https://${master02IP}:2380,${master03Name}=https://${master03IP}:2380"

keepaliveVer='2.0.18'
haproxyVer='2.0.6'

interface=$(netstat -nr | grep -E 'UG|UGSc' | grep -E '^0.0.0|default' | grep -E '[0-9.]{7,15}' | awk -F' ' '{print $NF}')
ipAddr=$(ip a s "${interface}" | sed -rn 's|.*inet ([0-9\.]{7,15})/[0-9]{2} brd.*$|\1|p')
peerName=$(hostname)
```

# Tools Setup
## cfssl & cfssljson

```bash
$ sudo bash -c "curl -o /usr/local/bin/cfssl ${cfsslDownloadUrl}/cfssl_linux-amd64"
$ sudo bash -c "curl -o /usr/local/bin/cfssljson ${cfsslDownloadUrl}/cfssljson_linux-amd64"
$ sudo chmod +x /usr/local/bin/cfssl*
```

## etcd

```bash
$ curl -sSL ${etcdDownloadUrl}/${etcdVer}/etcd-${etcdVer}-linux-amd64.tar.gz \
    | sudo tar -xzv --strip-components=1 -C /usr/local/bin/
```

## keepalived 

- Installation
    ```bash
    $ mkdir -p ~/temp
    $ sudo mkdir -p /etc/keepalived/

    $ curl -fsSL ${keepaliveDownloadUrl}/keepalived-${keepaliveVer}.tar.gz \
       | tar xzf - -C ~/temp

    $ pushd .
    $ cd ~/temp/keepalived-${keepaliveVer}
    $ ./configure && make
    $ sudo make install
    $ sudo cp keepalived/keepalived.service /etc/systemd/system/
    $ popd
    ```
- Configuration
    - with haproxy

        ```bash
        $ sudo bash -c 'cat > /etc/keepalived/keepalived.conf' << EOF
        ! Configuration File for keepalived

        global_defs {
          router_id LVS_DEVEL
        }

        vrrp_script check_haproxy {
          script "killall -0 haproxy"
          interval 3
          weight -2
          fall 10
          rise 2
        }

        vrrp_instance VI_1 {
          state MASTER
          interface ${interface}
          virtual_router_id 51
          priority 50
          advert_int 1
          authentication {
            auth_type PASS
            auth_pass 35f18af7190d51c9f7f78f37300a0cbd
          }
          virtual_ipaddress {
            ${virtualIP}
          }
          track_script {
            check_haproxy
          }
        }
        EOF
        ```
    - without haproxy

        ```bash
        $ sudo bash -c 'cat > /etc/keepalived/keepalived.conf' << EOF
        ! Configuration File for keepalived
        global_defs {
          router_id LVS_DEVEL
        }
        vrrp_script check_apiserver {
          script "/etc/keepalived/check_apiserver.sh"
          interval 3
          weight -2
          fall 10
          rise 2
        }
        vrrp_instance VI_1 {
          state MASTER
          interface ${interface}
          virtual_router_id 51
          priority 101
          authentication {
            auth_type PASS
            auth_pass 4be37dc3b4c90194d1600c483e10ad1d
          }
          virtual_ipaddress {
            ${virtualIP}
          }
          track_script {
            check_apiserver
          }
        }
        EOF

        $ sudo bash -c 'cat > /etc/keepalived/check_apiserver.sh' << EOF
        #!/bin/sh
        errorExit() {
          echo "*** \$*" 1>&2
          exit 1
        }
        curl --silent                           \
             --max-time 2                       \
             --insecure https://localhost:6443/ \
             -o /dev/null                       \
            || errorExit 'Error GET https://localhost:6443/'

        if ip addr | grep -q ${virtualIpAddr}; then
            curl --silent                                  \
                 --max-time 2                              \
                 --insecure https://${virtualIpAddr}:6443/ \
                 -o /dev/null                              \
                 || errorExit "Error GET https://${virtualIpAddr}:6443/"
        fi
        EOF
        ```

## haproxy 2.0.6
- install haproxy from source code

    ```bash
    $ curl -fs -O http://www.haproxy.org/download/$(echo ${haproxyVer%\.*})/src/haproxy-${haproxyVer}.tar.gz
    ```

## helm

## docker
