+++
title= "The Death of `externalIPs`: Migrating to Cilium Node IPAM"
date= 2026-04-05T00:31:58+03:00
author = ""
authorTwitter = "" #do not include @
tags = ["kubernetes", "cilium", "networking", "bare-metal", "homelab", "ipv6"]
keywords = ["Kubernetes v1.36", "externalIPs deprecation", "Cilium Node IPAM", "CVE-2020-8554", "LoadBalancer", "bare-metal Kubernetes", "self-hosted Kubernetes", "LoadBalancer service", "Cilium CNI", "externalIPs migration", "Kubernetes networking", "homelab Kubernetes", "IPv6 Kubernetes"]
description = "Kubernetes v1.36 is deprecating `service.spec.externalIPs`. If you're running a bare-metal or self-hosted cluster with public IPs tied to specific nodes, here's how I migrated to Cilium Node IPAM with minimal changes."
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
If you've been running a bare-metal or self-hosted Kubernetes cluster with public IPs attached to specific nodes, you've probably been using `service.spec.externalIPs`. It's a simple way to expose an application using the public IP of a node. For those who haven't used it, when network traffic arrives into the cluster, with the external IP (as destination IP) and the port matching that Service, rules and routes that Kubernetes has configured ensure that the traffic is routed to one of the endpoints for that Service.

Well, the clock has finally run out. Kubernetes v1.36 (shipping April 2026) deprecates `service.spec.externalIPs`, with full removal planned for v1.43. The reason is a real security issue ([CVE-2020-8554](https://nvd.nist.gov/vuln/detail/cve-2020-8554)) where any user with permission to create Services could specify arbitrary external IPs and intercept traffic. For multi-tenant clusters, that's a legitimate concern. For those of us running single-admin single-tenant setups, it feels like mostly noise. But the field is going away regardless, so it's worth figuring out the replacement before you're forced to.

The two replacements you'll find everywhere are MetalLB and Cilium's LB IPAM. Both are great tools, but they're designed for a different situation: one where *you* own a range of IP addresses that you can hand out and advertise via ARP or BGP. If your IPs come from a cloud provider and are permanently routed to a specific VM's NIC, neither of these fits. MetalLB's L2 mode sends ARP/NDP announcements that the cloud provider's network will simply ignore, since the routing is already handled at the infrastructure level. Cilium's LB IPAM (`CiliumLoadBalancerIPPool`) by default allocates at most one IPv4 and one IPv6 per service, so if you have multiple nodes with different public IPv6 addresses that should all serve the same service, you will need to define in an annotation all the IPs you want to be handed to this service. But in our case, the IPs are a property of specific nodes.

This is where I discovered a Cilium feature called Node IPAM LoadBalancer (`loadBalancerClass: io.cilium/node`), and it seems that it's exactly what `externalIPs` was doing, just with a supported mechanism.

Instead of declaring which IPs a Service should listen on, Cilium reads the addresses from the nodes themselves and automatically populates `status.loadBalancer.ingress`. When a new node joins with a public IP, it appears. When a node is removed, it disappears. And all of this without having to manually adjust an annotation in all of your services. A good rule of thumb to choose between LB IPAM and Node IPAM: if you find yourself manually typing IP addresses into annotations, you're probably looking at the wrong feature.

Node IPAM is disabled by default, so you'll need to add this to your Cilium Helm values and upgrade:
```yaml
nodeIPAM:
  enabled: true
```

After upgrading, restart the cilium-operator
```bash
kubectl rollout restart deployment cilium-operator -n kube-system
```

Now we are ready to begin our migration.

So a service that used to look like this:
```yaml
spec:
  type: LoadBalancer
  externalIPs:
    - 198.51.100.1  # node-a IPv4
    - 2001:db8::1   # node-a IPv6
    - 2001:db8::2   # node-b IPv6
```

becomes this:
```yaml
spec:
  type: LoadBalancer
  loadBalancerClass: io.cilium/node
  externalTrafficPolicy: Cluster
  ipFamilies:
    - IPv6
    - IPv4
  ipFamilyPolicy: PreferDualStack
```

You can also add an annotation to restrict which nodes get selected, otherwise every node in the cluster gets advertised, effectively controlling which node exposes which service:
```yaml
metadata:
  annotations:
    io.cilium.nodeipam/match-node-labels: "ingress-ready=true"
```

Then label the nodes that have public IPs and you want to serve traffic for the service:
```bash
kubectl label node node-a ingress-ready=true
kubectl label node node-b ingress-ready=true
```

That's it. Cilium handles the rest.

Node IPAM keeps the simplicity and the same mental model as `externalIPs` while fitting into how Kubernetes actually expects LoadBalancer services to work. If you're currently using `externalIPs`, especially with Cilium as your CNI, this is the migration that requires the fewest moving parts.

