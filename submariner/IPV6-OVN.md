
# **Submariner Enhancement for IPv6 Support**

## **Summary**
This proposal outlines the changes required in Submariner for  OVN Kubernetes CNI to enable IPv6 support, ensuring seamless connectivity between clusters using Submariner. 
The main proposal has the full design https://github.com/submariner-io/enhancements/blob/devel/submariner/IPV6-datapath.md. This covers only the OVN CNI part of it.

---

## Design Details

The OVNKubernetes handler programs network policies and routes to direct traffic from the gateway and non-gateway nodes to direct the traffic to the remote cluster.
At present the routes are only programmed for IPv4 for addresses. We need to enhance this to support IPV6 addresses as well.

The handler for creating the Gateway and NonGateway routes needs to be enhanced.

### GatewayRoute CRD:

The GatewayRoute will now populate Ipv4 and Ipv6 addresses for the next hops and remote CIDRs for a dual-stack environment. For Ipv6 only it uses just the IPv6 address
for these fields. Ipv4 will continue as it is.

The next hop will be the interface IP of ovn-k8s-mp0 interface, which is expected to have two IPs in the case of dual-stack environments.


```yaml
apiVersion: submariner.io/v1alpha1
kind: GatewayRoute
metadata:
  name: remote-cluster-route
spec:
  nextHops:
    - "fd00:abcd::1"
    - "192.168.1.1"
  remoteCIDRs:
    - "fd00:4321::/64"
    - "10.0.0.0/8"
```

### NonGatewayRoute CRD:

The NonGatewayRoute will follow the same pattern as GatewayRoute for populating next hops and remotecidrs.

The nexthops will be the transit switch IP of the gateway node.

#### **NonGatewayRoute CRD Example**

```yaml
apiVersion: submariner.io/v1alpha1
kind: NonGatewayRoute
metadata:
  name: non-gw-route
spec:
  nextHops:
    - "fd00:cafe::1"
    - "172.16.0.1"
  remoteCIDRs:
    - "fd00:5678::/64"
    - "192.168.0.0/16"
```


### GatewayRoute Handler

The gateway route handler should iterate through the ips configured and must identify the ipv4 remote CIDR and nextHop and Ipv6 remote CIDR pairs from the gateway route.
It will program one or two logical route policies and logical routes based on the network configuration. For dual-stack, it will be two LRPs. Similarly a logical route will
be added to route the traffic from the non-gateway nodes and will be resubmitted to hit the above added Logical route policy.

```plaintext
match: "ip6.dst==fd00:5678::/64"
action: reroute
nexthops: ["fd00:abcd::1"]
priority: 20000
```

```plaintext
destination: "fd00:1234::/64"
nexthop: "fd00:cafe::1"
priority: 200
```

### NonGatewayRoute Handler

The NonGatewayRoute  handler should iterate through the ips configured and must identify the ipv4 remote CIDR and nextHop and Ipv6 remote CIDR pairs from the gateway route.
It will program one or two logical route policies and logical routes based on the network configuration. For dual stack it will be two LRPs.

```plaintext
match: "ip6.dst==fd00:5678::/64"
action: reroute
nexthops: ["fd00:abcd::1"]
priority: 20000
```

---
