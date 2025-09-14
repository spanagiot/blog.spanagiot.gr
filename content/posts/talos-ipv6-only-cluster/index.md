+++
title = "Building an IPv6-Only Kubernetes Cluster with Talos and talhelper"
date = "2025-09-14T13:45:11+03:00"
tags = ["IPv6", "Kubernetes", "Talos", "Talos Linux", "talhelper", "homelab", "Cilium"]
keywords = ["Kubernetes IPv6", "IPv6-only cluster", "Talos Linux", "Talos", "talhelper", "homelab Kubernetes", "Cilium CNI", "DNS64 NAT64"]
description = "See how I run a Kubernetes cluster exclusively on IPv6 using Talos Linux and talhelper. The post walks through setup, networking considerations, and DNS64/NAT64 configuration. It’s a starting point for experimenting with IPv6 networking in Kubernetes."
+++

For many years now I've been experimenting with Kubernetes and ways to host it in the machines I have laying around in my homelab. This includes bare-metal machines, primarily Lenovo Thinkcentres (the famous 1L computers), Raspberry Pis, but also virtual machines on my Proxmox hypervisors.

My first self-hosted Kubernetes installation was a k3s cluster using Raspberry Pis. I used the management interface by Rancher and had created a bunch of clusters using `cattle`, their (now deprecated) container orchestration system. So seeing a lightweight Kubernetes distribution built specifically with edge devices in-mind (exactly what a Raspberry Pi is considered) from a company that had built tools I used and trusted made my choice easier. At that time I just wanted a running cluster for experimenting with how to deploy a couple of workloads in Kubernetes and to avoid completely the details of configuring in depth that cluster, something that k3s was built for. Combined with [k3sup](https://github.com/alexellis/k3sup), creating k3s clusters took literally 2 minutes and one command per node.

I stayed with k3s for a long time (and I still consider it my go to when someone asks for a quick way to spin up a cluster) until something called `Talos` caught my attention. [Talos](https://www.talos.dev/) is a minimal Linux distribution designed to run Kubernetes, and just that. You don't get any SSH or console access to the machine and every interaction happens through an API. This reduces the operational overhead of having to manage the underlying OS of the nodes and the system configuration is a YAML file that can be stored to git, ensuring a predictable state. I really like managing my Kubernetes workloads using GitOps, where every manifest is pulled from a git repo and applied to my cluster automatically, so extending this to my nodes as well was very tempting.

With the Kubernetes distribution chosen, the next challenge was networking. Instead of going with a more traditional, IPv4 only, or a more progressive dual-stack cluster, I wanted to go all-in on IPv6. An IPv6 only cluster simplifies things by eliminating the overhead of NAT, avoids issues with overlapping private IP ranges (something that bit me in the past with Docker and their `172.17.0.0/16` subnet), improves my knowledge of IPv6 and how it's handled by CNI plugins and also makes it a great topic for future posts :)

As I stated earlier, I'm going to use Talos as my OS and Kubernetes distribution. To get a Talos image you can use the [Talos Linux Image Factory](https://factory.talos.dev/). There, you can specify the hardware type, the CPU architecture and system extensions, such as kernel module driver for BTRFS,iSCSI tools etc.

To make Talos configuration simpler and easier, I'm using a very nice tool called [talhelper](https://budimanjojo.github.io/talhelper/latest/). This tool abstracts some parts of the Talos configuration and (more importantly) supports natively [SOPS](https://github.com/getsops/sops) so you can commit its files directly to git, something currently not possible with the native Talos CLI tool. If we didn't use this, we would need to store the cluster certificates and private keys somewhere safe but also accessible, as we couldn't perform any operation to the cluster without them.

I will skip the part of installing and configuring talhelper and SOPS, their [documentation](https://budimanjojo.github.io/talhelper/latest/getting-started/#before-you-begin) has all the information you need to get started. Also, this post assumes that you know to create a Talos cluster, apply the configuration files and bootstrap it. If not, they also have a detailed [documentation](https://www.talos.dev/v1.11/introduction/getting-started/) to help you.

Now, let's dive into the configuration!

This is my working `talconfig.yaml`

```yaml
---
clusterName: homeprod
endpoint: https://[2001:db8::be24:11ff:fe97:9748]:6443
nodes:
  - hostname: talos-ipv6-only-01
    controlPlane: true
    ipAddress: 2001:db8::be24:11ff:fe97:9748
    installDisk: /dev/sda
    nameservers:
      - 2606:4700:4700::64
      - 2606:4700:4700::6400
patches:
  - |-
    cluster:
      network:
        podSubnets:
          - 2001:db8:8::/64
        serviceSubnets:
          - 2001:db8:1::/112
    machine:
      kubelet:
        nodeIP:
          validSubnets:
            - 2001:db8::/32
      network:
        extraHostEntries:
          - ip: 2001:db8::be24:11ff:fe97:9748
            aliases:
              - talos-ipv6-only-01
```

The first needed change is to add nameservers that support DNS64 (and you must have NAT64 gateway in your network). Here, I'm using the DNS64 servers provided by Cloudflare, but you can you any other DNS64 servers you want. To briefly explain why this is needed, many external services are still IPv4-only, making them unreachable from an IPv6-only cluster. DNS64 synthesizes these IPv4 addresses into IPv6 addresses using a NAT64 prefix, such as `64:ff9b::/96`, allowing IPv4-only domains to be resolved. The NAT64 gateway then translates these addresses back to IPv4 and forwards them to their intended destination.

```yaml
    nameservers:
      - 2606:4700:4700::64
      - 2606:4700:4700::6400
```

Talos [doesn't use the nameservers provided with ND](https://github.com/siderolabs/talos/issues/9372) and the default nameservers they use are `1.1.1.1` and `8.8.8.8`, so you will definitely need to add IPv6 nameservers. Also, don't be confused if you see in the logs that the node can't communicate with either `1.1.1.1` or `8.8.8.8`. If you don't set any IPv4 nameservers, Talos will automatically set them for you.

In the cluster patches, we define the subnets kubernetes will use for pods and services. For the service subnet, I always choose the subnet to be at most a `/112` because when I tried going with a `/64`, or anything larger than `/112` the kubelet service refused to start stating that the subnet is too large. I can't find anything on the documentation for this behavior and I don't know if this has been fixed since `/112` is pretty large for my needs so I didn't bother rechecking.

The validSubnets field configures the networks to pick kubelet node IP from, so kubelet won't pick any other IP, or IPv4 if the machine has any.

Finally, if the hostname of your machine can't be resolved from the Google DNS servers, you will need to add it as an extra host entry for the OS to be able to resolve it. You will also need to add the hostnames of all the other nodes you will add in your cluster in the future.

That’s it. This is the working base configuration I use to run my Talos-based Kubernetes cluster exclusively on IPv6. Going IPv6-only with Talos turned out to be simpler than I expected once DNS64/NAT64 was in place. The biggest win is a cleaner network without NAT, and Kubernetes just works. If you try this yourself, watch your service subnet sizing and don’t forget to set proper IPv6 nameservers.

Next up, I want to talk about how Cilium CNI behaves in this IPv6-only world.
