---
layout: post
title:  setup 1.15.3 HA kubernetes Cluster Playbook (basic environment)
date:   2019-09-25 16:47:24 +0800
categories: ["kubernetes"]
---

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Extended etcd Topology](#extended-etcd-topology)
  - [Kubernetes HA cluster with external etcd](#kubernetes-ha-cluster-with-external-etcd)
  - [Server Matrix](#server-matrix)
    - [List](#list)
    - [`/etc/hosts`](#etchosts)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Extended etcd Topology
## [Kubernetes HA cluster with external etcd](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#external-etcd-topology)

<img src="{{site.url}}/images/external-etcd-topology.png" style="width: 666px;" />

## Server Matrix

### List

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
