---
layout: post
title:  1.15.3 HA kubernetes Cluster Playbook (basic environment)
date:   2019-09-25 16:47:24 +0800
categories: ["kubernetes"]
tags: ["kuberentes"]
excerpt_separator: <!--more-->
---

## Extended etcd Topology
### [Kubernetes HA cluster with external etcd](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#external-etcd-topology)

<img src="{{site.url}}/assets/images/external-etcd-topology.png" style="width: 666px;" />

<!--more-->
### Server Matrix

#### Environment List

| Hostname   | IP Address      | etcd              | cfssl & cfssljson | keepalived | haproxy  |
|------------|-----------------|-------------------|-------------------|------------|----------|
| master01   | 192.168.100.200 | master01 (etcd-0) | &#x2714;          | &#x2714;   | &#x2714; |
| master02   | 192.168.100.201 | master02 (etcd-1) | &#x2714;          | &#x2714;   | &#x2714; |
| master03   | 192.168.100.202 | master03 (etcd-2) | &#x2714;          | &#x2714;   | &#x2714; |
| node01     | 192.168.100.203 | &#x2717;          | &#x2717;          | &#x2717;   | &#x2717; |
| node02     | 192.168.100.204 | &#x2717;          | &#x2717;          | &#x2717;   | &#x2717; |
|            |                 |                   |                   |            |          |
| virtual IP | 192.168.100.250 |                   |                   |            |          |
|------------|-----------------|-------------------|-------------------|------------|----------|
{:.inner-borders}


#### `/etc/hosts`

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

#### variables
<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b>execute the variables in all console (masters) at the very begining, make sure all servers are using the exact same value (and avoid manual input)
</div>

```bash
## change if necessary
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
helmVer='v2.14.3'

interface=$(netstat -nr | grep -E 'UG|UGSc' | grep -E '^0.0.0|default' | grep -E '[0-9.]{7,15}' | awk -F' ' '{print $NF}')
ipAddr=$(ip a s "${interface}" | sed -rn 's|.*inet ([0-9\.]{7,15})/[0-9]{2} brd.*$|\1|p')
peerName=$(hostname)
```

## Tools Setup
### cfssl & cfssljson
<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b> cfssl and cfssljson need to be setup in all masters!
</div>

```bash
$ sudo bash -c "curl -o /usr/local/bin/cfssl ${cfsslDownloadUrl}/cfssl_linux-amd64"
$ sudo bash -c "curl -o /usr/local/bin/cfssljson ${cfsslDownloadUrl}/cfssljson_linux-amd64"
$ sudo chmod +x /usr/local/bin/cfssl*
```

### etcd
<div class="alert alert-success" role="alert">
<i class="fa fa-check-square-o"></i>
<b>Tip: </b> etcd need to be setup in all masters!
</div>

```bash
$ curl -sSL ${etcdDownloadUrl}/${etcdVer}/etcd-${etcdVer}-linux-amd64.tar.gz \
    | sudo tar -xzv --strip-components=1 -C /usr/local/bin/
```

### keepalived

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
    $ rm -rf ~/temp
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
          priority 50
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

- start keepalived serice and verify
    ```bash
    $ sudo systemctl enable keepalived.service
    $ sudo systemctl start keepalived.service

    $ sudo systemctl is-enabled keepalived.service
    enabled
    $ sudo systemctl is-active keepalived.service
    active

    $ ip a s ${interface}
    2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 00:50:50:85:96:64 brd ff:ff:ff:ff:ff:ff
        inet 192.168.100.200/24 brd 192.168.100.255 scope global noprefixroute eno1
           valid_lft forever preferred_lft forever
        inet 192.168.100.250/32 scope global eno1
           valid_lft forever preferred_lft forever
        inet6 fe80::250:fe85:86ff:9624/64 scope link
           valid_lft forever preferred_lft forever
    ```

    <div class="alert alert-success" role="alert">
    <i class="fa fa-check-square-o"></i>
    <b>Tip: </b> One of the master will be setup to virutal dual networking card and show 2 ip addresses. The one without Broadcast is the virutal IP.
    </div>

### Haproxy 2.0.6
- Install haproxy from source code
    ```bash
   $ curl -fsSL http://www.haproxy.org/download/$(echo ${haproxyVer%\.*})/src/haproxy-${haproxyVer}.tar.gz \
           | tar xzf - -C ~

    $ pushd .
    $ cd ~/haproxy-${haproxyVer}
    $ make TARGET=linux-glibc \
           USE_LINUX_TPROXY=1 \
           USE_ZLIB=1 \
           USE_REGPARM=1 \
           USE_PCRE=1 \
           USE_PCRE_JIT=1 \
           USE_OPENSSL=1 \
           SSL_INC=/usr/include \
           SSL_LIB=/usr/lib \
           ADDLIB=-ldl \
           USE_SYSTEMD=1
    $ sudo make install
    $ sudo cp haproxy /usr/sbin/
    $ sudo cp examples/haproxy.init /etc/init.d/haproxy && sudo chmod +x $_
    $ popd
    $ rm -rf ~/haproxy-${haproxyVer}
    ```

- Configure Haproxy
    ```bash
    $ sudo bash -c 'cat /etc/haproxy/haproxy.cfg' << EOF
    #---------------------------------------------------------------------
    # Example configuration for a possible web application.  See the
    # full configuration options online.
    #
    #   http://haproxy.1wt.eu/download/2.0/doc/configuration.txt
    #
    #---------------------------------------------------------------------

    #---------------------------------------------------------------------
    # Global settings
    #---------------------------------------------------------------------
    global
        # to have these messages end up in /var/log/haproxy.log you will
        # need to:
        #
        # 1) configure syslog to accept network log events.  This is done
        #    by adding the '-r' option to the SYSLOGD_OPTIONS in
        #    /etc/sysconfig/syslog
        #
        # 2) configure local2 events to go to the /var/log/haproxy.log
        #   file. A line like the following can be added to
        #   /etc/sysconfig/syslog
        #
        #    local2.*                       /var/log/haproxy.log
        #
        log         127.0.0.1 local2

        chroot      /var/lib/haproxy
        pidfile     /var/run/haproxy.pid
        maxconn     4000
        user        haproxy
        group       haproxy
        daemon

        # turn on stats unix socket
        stats socket /var/lib/haproxy/stats

    #---------------------------------------------------------------------
    # common defaults that all the 'listen' and 'backend' sections will
    # use if not designated in their block
    #---------------------------------------------------------------------
    defaults
        mode                    http
        log                     global
        option                  httplog
        option                  dontlognull
        option http-server-close
        option forwardfor       except 127.0.0.0/8
        option                  redispatch
        retries                 3
        timeout http-request    10s
        timeout queue           1m
        timeout connect         10s
        timeout client          1m
        timeout server          1m
        timeout http-keep-alive 10s
        timeout check           10s
        maxconn                 3000

    #---------------------------------------------------------------------
    # kubernetes apiserver frontend which proxys to the backends
    #---------------------------------------------------------------------
    frontend kubernetes-apiserver
        mode                 tcp
        bind                 *:16443
        option               tcplog
        default_backend      kubernetes-apiserver

    #---------------------------------------------------------------------
    # round robin balancing between the various backends
    #---------------------------------------------------------------------
    backend kubernetes-apiserver
        mode        tcp
        balance     roundrobin
        option      tcplog
        option      tcp-check
        server      ${master01Name} ${master01IP}:6443 check
        server      ${master02Name} ${master02IP}:6443 check
        server      ${master03Name} ${master03IP}:6443 check

    #---------------------------------------------------------------------
    # collection haproxy statistics message
    #---------------------------------------------------------------------
    listen stats
        bind                 :8000
        stats auth           admin:devops
        maxconn              50
        stats refresh        10s
        stats realm          HAProxy\ Statistics
        stats uri            /healthy
    EOF
    ```

- Configure Service
    ```bash
    $ sudo bash -c 'cat > /lib/systemd/system/haproxy.service' << EOF
    [Unit]
    Description=HAProxy Load Balancer
    After=network.target syslog.service
    Wants=syslog.service

    [Service]
    Environment="CONFIG=/etc/haproxy/haproxy.cfg" "PIDFILE=/run/haproxy.pid"
    EnvironmentFile=-/etc/default/haproxy
    ExecStartPre=/usr/sbin/haproxy -f $CONFIG -c -q
    ExecStart=/usr/sbin/haproxy -W -f $CONFIG -p $PIDFILE $EXTRAOPTS
    ExecReload=/usr/sbin/haproxy -f $CONFIG -c -q $EXTRAOPTS $RELOADOPTS
    ExecReload=/bin/kill -USR2 $MAINPID
    KillMode=mixed
    Restart=always
    Type=forking

    # The following lines leverage SystemD's sandboxing options to provide
    # defense in depth protection at the expense of restricting some flexibility
    # in your setup (e.g. placement of your configuration files) or possibly
    # reduced performance. See systemd.service(5) and systemd.exec(5) for further
    # information.

    # NoNewPrivileges=true
    # ProtectHome=true
    # If you want to use 'ProtectSystem=strict' you should whitelist the PIDFILE,
    # any state files and any other files written using 'ReadWritePaths' or
    # 'RuntimeDirectory'.
    # ProtectSystem=true
    # ProtectKernelTunables=true
    # ProtectKernelModules=true
    # ProtectControlGroups=true
    # If your SystemD version supports them, you can add: @reboot, @swap, @sync
    # SystemCallFilter=~@cpu-emulation @keyring @module @obsolete @raw-io

    [Install]
    WantedBy=multi-user.target
    EOF
    ```
- Start Service and Verify
    ```bash
    $ sudo systemctl enabled haproxy.service
    $ sudo systemctl start haproxy.service

    $ sudo systemctl is-enabled haproxy.service
    enabled
    $ sudo systemctl is-active haproxy.service
    active
    ```

- Result
<img src="{{site.url}}/assets/images/haproxy.png" style="width: 999px;" />

### helm
- Installation
    ```bash
    $ curl -fsSL \
        https://get.helm.sh/helm-v2.14.3-linux-amd64.tar.gz \
        | sudo tar -xzv --strip-components=1 -C /usr/local/bin/

    $ while read -r _i; do
        sudo chmod +x "/usr/local/bin/${_i}"
    done < <(echo helm tiller)
    ```
- Configration
    ```bash
    $ helm init
    $ helm init --client-only

    $ kubectl create serviceaccount -n kube-system tiller
    $ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
    $ kubectl patch deploy -n kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

    $ helm repo add jetstack https://charts.jetstack.io
    ```

### docker
- Presetup
    ```bash
    # clean environment
    $ sudo yum remove -y docker \
                       docker-client \
                       docker-client-latest \
                       docker-common \
                       docker-latest \
                       docker-latest-logrotate \
                       docker-logrotate \
                       docker-selinux \
                       docker-engine-selinux \
                       docker-engine
    # nice to have
    $ sudo yum -y groupinstall 'Development Tools'

    $ sudo yum install -y yum-utils \
                        device-mapper-persistent-data \
                        lvm2 \
                        bash-completion*

    $ sudo yum-config-manager \
         --add-repo \
         https://download.docker.com/linux/centos/docker-ce.repo
    $ sudo yum-config-manager --disable docker-ce-edge
    $ sudo yum-config-manager --disable docker-ce-test
    $ sudo yum makecache
    ```

- Installaiton
    ```bash
    $ dockerVer=$(sudo yum list docker-ce --showduplicates \
                    | sort -r \
                    | grep 18\.09 \
                    | awk -F' ' '{print $2}' \
                    | awk -F':' '{print $NF}' \
                )
    $ sudo yum install -y \
             docker-ce-${dockerVer}.x86_64 \
             docker-ce-cli-${dockerVer}.x86_64 \
             containerd.io
    ```
- Configuraiton
    ```bash
    $ sudo systemctl enable --now docker
    $ sudo systemctl status docker
    $ sudo chown -a -G docker $(whomai)
    ```
