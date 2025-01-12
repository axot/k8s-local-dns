[![GitHub release](https://img.shields.io/github/release/colopl/k8s-local-dns.svg)](https://github.com/colopl/k8s-local-dns/releases)
[![Go Report Card](https://goreportcard.com/badge/github.com/colopl/k8s-local-dns)](https://goreportcard.com/report/github.com/colopl/k8s-local-dns)
[![Travis Result](https://travis-ci.com/colopl/k8s-local-dns.svg?token=VGvwH2B72tty9c4FwZGr&branch=master)](https://travis-ci.com/colopl/k8s-local-dns.svg?token=VGvwH2B72tty9c4FwZGr&branch=master)

## Overview
Using Kubernetes' node cache should solve most DNS issues, there are few reasons why we built a custom version here.

1. Got OOM killed in our stress test.
2. Old Kubernetes couldn't use it.
3. Hard to control its behavior specially if you are using management Kubernetes like GKE.

We added all CoreDNS addons, default to filter AAAA records and more.

## Known issues
- Daemonset rolling update will cause DNS query timeout for few seconds.

## DNS releated issues without this

- [DNS lookup timeouts due to races in conntrack](https://github.com/weaveworks/weave/issues/3287)
- [DNS latency of 5s when uses iptables forward in pods network traffic](https://github.com/kubernetes/kubernetes/issues/62628)
- [DNS intermittent delays of 5s](https://github.com/kubernetes/kubernetes/issues/56903)

## Warning: Network policy

If running with network policy, please see the [README for
node-local-dns](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns/nodelocaldns#network-policy-and-dns-connectivity);
network policy will likely need to be configured for the node-local-dns agent.

## Other caveats

* If the local-dns process crashes, DNS resolution will not function on that
  node until it is restarted.

## How it works

Normally the local-dns agent:
* uses an unused IP (typically `169.254.20.10`)
* the kubelet `--cluster-dns` flag is used to specify that pods should use that
  IP address (`169.254.20.10`) as their DNS server
* CoreDNS runs as a daemonset on every node, configured to listen to the
  internal IP (`169.254.20.10`)
* The local-dns agent configures IP tables rules to avoid conntrack / NAT

In this mode, we instead intercept the existing kube-dns service IP a few
things:
* We configure the local-dns agent to intercept the kube-dns service IP
* kubelet is already configured to send queries to that service, by default
* When the local-dns agent configures the kube-dns service IP to avoid
  conntrack/NAT, this takes precedence over the normal DNS service routing.

## Installation

A script is provided, simply run `cd ./hack && cp configmap.yaml.sample configmap.yaml && ./install.sh`

## Removal

Removal is more complicated that installation.  We can remove the daemonset, and as
part of pod shutdown the local-dns cache should remove the IP interception
rules.  However, if something goes wrong with the removal, the IP interception rules
will remain in place, but the local-dns cache will not be running to serve
the intercepted traffic, and DNS lookup will be broken on that node.  However,
restarting the machine will remove the IP interception rules, so if this is done
as part of a cluster update the system will self-heal.

The procedure therefore is:

* Run `cd ./hack && ./uninstall.sh`
* Upgrade cluster

## References
* https://github.com/justinsb/dns/tree/standalone_nodecache/tools/node-cache
* https://github.com/kubernetes/dns/tree/master/cmd/node-cache
* https://github.com/zalando-incubator/kubernetes-on-aws/blob/dev/cluster/manifests/coredns-local