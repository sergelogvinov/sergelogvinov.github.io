# sergelogvinov.github.io

About me.

## Languages

* golang
* python
* ruby
* php
* asm
* c/c++
* pascal
* bash/sh

## Kubernetes world

* hybrid/multi cluster, bare metal + cloud
* kubernetes from scratch (ansible roles)
* cni - cilium, wavenet, kube-router, kilo
* fluent-bit/fluentd + plugin, loki, clickhouse
* grafana, prometheus, custom exporters (golang, python)
* ingress-nginx, gloo, haproxy, traefic, skipper
* helm + sops, fluxcd, ansible roles

## Linux world

* linux auto install (automated installation)
* puppet + hiera + activemq (~60 modules + 2 ruby libs)
* ansible (~40 roles)
* PRs to foreman project
* virtualisation xen,kvm with vt-d and numa optimization.
* lxc with pre-built templates. Like the docker but only one layer. (deploy system)
* openstack from scratch using puppet + one network plugin.
* AWS, Azure, GCP, Oracle, Digitalocean, Hetzner, Ovh, Scaleway, Upcloud and ~10 others
* terraform

## OS

* debian + build deb packages
* ubuntu
* sles as VM hypervisor
* centos
* openbsd
* freebsd
* openwrt (custom firmware)

## Network

* firewalls - iptables + ipsets, pf
* cisco switces - acls, vlans, port channels
* bgp - bird for load balancing
* bind9, powerdns - primary/secondary/geo view
* gateways - linux/openbsd/openwrt, multi wan lb
* openvpn, ipsec, wireguard

## Database

* postgres from 8.4 and stored procedures (plsql)
* clickhouse
* redis, keyDB
* mongodb
* rabbitmq
* influx
* memcache

## CI/CD

* teamcity
* github actions
* heroku
* jenkins
* makefile
* dockerfile

## Cryptocurrency

Private solution like infura, main/testnets env.

* ethereum (one smart contract)
* bitcoin
* waves
* ergo

## Solutions for offices (SAS)

* work time accounting system
* Network gateways, nat, web proxy with filtering, website blocker
    * openbsd as router (read only root fs)
    * primary/secondary dns (bind9)
    * squid + filtering
    * mail server  (sendmail + sasl, sendmail filters m4) + pop3/imap server (dovecot)
    * tftp/dhcp boot + unattended windows install (it takes about 30 minute to full preparation windows workstation, no system administrator required)
* office workstations (based on ubuntu)
* automation external management for linux like puppet/chef but uses track (python) and python daemons on the workstations. Daemons receive the jobs from the Track system, launch it and put the result to the issue.

## University time 1998-2004

* microchip PIC (16 bit) home automatisation (asm)
* network hardware 10Mbit bidwand, error rate, packet collisions (windows application uses libpcap)
* home ISP, gateways, firewalls, traffic billing, pptp/pppoe server (for windows clients)
* high performed file server (samba with optimisation) + journal file system (samba virtual file system) and business logic around it. (freebsd)

## Collage time 1996-1998

* dos game like arkanoid (pascal + asm injections)
