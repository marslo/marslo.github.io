---
layout: post
title:  1.15.3 HA kubernetes Cluster Playbook (load balance)
date:   2019-09-30 21:38:45 +0800
categories: ["kubernetes", "centos"]
tags: ["kuberentes", "centos"]
excerpt_separator: <!--more-->
---

## Objective
<img src="{{site.url}}/assets/images/external-etcd-topology-lb.png" style="width: 666px;" />

<!--more-->

### variables
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
ipAddr=$(ip a s "${interface}" | sed -rn 's|\W*inet[^6]\W*([0-9\.]{7,15}).*$|\1|p')
peerName=$(hostname)
```

