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
- [Tools Installation](#tools-installation)
  - [cfssl & cfssljson](#cfssl--cfssljson)
  - [etcd](#etcd)
  - [keepalived](#keepalived)
  - [haproxy](#haproxy)
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
master01Name="master01"
master02Name="master02"
master03Name="master03"
master01IP="192.168.100.200"
master01IP="192.168.100.201"
master01IP="192.168.100.202"
virtualIP="192.168.100.250"

leadIP="${master01IP}"
leadName="${master01Name}"

k8sVer="v1.15.3"
etcdVer='v3.3.15'
keepaliveVer='2.0.18'
haproxyVer="2.0.6"

etcdPath='/etc/etcd/ssl

interface=$(netstat -nr | grep -E 'UG|UGSc' | grep -E '^0.0.0|default' | grep -E '[0-9.]{7,15}' | awk -F' ' '{print $NF}')
ipAddr=$(ip a s "${interface}" | sed -rn 's|.*inet ([0-9\.]{7,15})/[0-9]{2} brd.*$|\1|p')
peerName=$(hostname)
```

# Tools Installation
## cfssl & cfssljson

## etcd

## keepalived

## haproxy

## helm

## docker
