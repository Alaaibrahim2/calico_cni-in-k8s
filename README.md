# calico_cni-in-k8s
Calico CNI Deep Dive Lab
This repository provides a comprehensive, step-by-step walkthrough of how Calico CNI works in Kubernetes - from pod creation to inter-node communication.
Architecture Overview
Calico consists of several key components:
calico-node: DaemonSet running on each node (handles BGP, routes, and Felix)
calico-kube-controllers: Single deployment (handles IPAM cleanup, policy processing)
CNI Plugins:
/opt/cni/bin/calico (main CNI plugin)
/opt/cni/bin/calico-ipam (IPAM plugin)
Configuration: /etc/cni/net.d/10-calico.conflist
# Verify Calico components
kubectl get pods -n kube-system | grep calico
calico-kube-controllers-658d97c59c-zfp99   1/1     Running   45 (22m ago)   87d
calico-node-4c5z5                          1/1     Running   37 (23m ago)   87d
calico-node-kqfgr                          1/1     Running   37 (22m ago)   87d
calico-node-nn766                          1/1     Running   37 (22m ago)   87d
Pod Creation Flow
Step 1: Kubelet receives pod creation request
When you create a pod, the Kubernetes scheduler assigns it to a node, and kubelet on that node handles the creation.
# Create a test pod on node1
kubectl run test-pod --image=nginx --overrides='{"spec": {"nodeName": "node1"}}'
pod/test-pod created
Step 2: Kubelet communicates with CRI
Kubelet communicates with the Container Runtime Interface (CRI) via a Unix socket to create the pod sandbox.
containerd socket: /var/run/containerd/containerd.sock
CRI-O socket: /var/run/crio/crio.sock
Step 3: CRI creates pod sandbox
The CRI (containerd/CRI-O):
Creates a network namespace (/var/run/netns/cni-<id>)
Starts the pause container in this namespace
Step 4: Kubelet reads CNI configuration
Kubelet reads the CNI configuration file to determine which plugin to use:
cat /etc/cni/net.d/10-calico.conflist
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "nodename": "node1",
      "ipam": {
        "type": "calico-ipam"
      }
    }
  ]
}
Step 5: Kubelet executes CNI plugin
Kubelet executes the main CNI plugin:
/opt/cni/bin/calico
IP Allocation Process
Step 6: CNI plugin calls IPAM plugin
The CNI plugin calls the IPAM plugin using the standard CNI library:
/opt/cni/bin/calico-ipam
Step 7: IPAM plugin contacts Kubernetes API
The IPAM plugin directly contacts the Kubernetes API server (https://10.96.0.1:443) to:
Read IP pool settings
Allocate a new block if needed (192.168.166.128/26)
Assign an IP address (192.16.166.170)
Create a BlockAffinity record
Note: The IPAM plugin communicates directly with the Kubernetes API - it does NOT communicate with calico-node.
Step 8: Verify pod IP assignment
kubectl get pod test-pod -o wide
NAME       READY   STATUS    RESTARTS   AGE   IP                NODE    NOMINATED NODE   READINESS GATES
test-pod   1/1     Running   0          27s   192.168.166.170   node1   <none>           <none>
Step 9: Verify BlockAffinity
kubectl get blockaffinities.crd.projectcalico.org
NAME                       AGE
master-192-168-219-64-26   87d
node1-192-168-166-128-26   87d
node2-192-168-104-0-26     87d
Key distinction:
IPAMBlock: The actual IP block (192.168.166.128/26)
BlockAffinity: The record linking node1 to this block
Network Configuration
Step 10: CNI plugin configures pod networking
The CNI plugin:
Assigns the IP to the pod's network namespace: 192.168.166.170/32
Creates a veth pair between the pod namespace and host: cali7fba7a35b74
Adds a local route on the host
# Check local route on node1
ip route show | grep 192.168.166.170
192.168.166.170 dev cali7fba7a35b74 scope link
Step 11: Calico-node monitors changes
Calico-node watches the Kubernetes API for changes to:
BlockAffinities
IPAMBlocks
NetworkPolicies
It uses this information to build its internal routing table.
Step 12: BGP peering establishment
Calico-node establishes BGP sessions with other nodes:
# Check BGP connections on node1
sudo ss -tn | grep ':179'
ESTAB 0      0               172.31.34.115:179            172.31.40.234:43559       
ESTAB 0      0               172.31.34.115:179             172.31.35.22:56249
This shows BGP sessions established between node1 (172.31.34.115) and:
master (172.31.40.234)
node2 (172.31.35.22)
Step 13: Route distribution
Calico-node adds routes to the host's routing table for blocks owned by other nodes:
# Check inter-node routes on node1
ip route show | grep tunl0
192.168.104.0/26 via 172.31.35.22 dev tunl0 proto bird onlink 
192.168.219.64/26 via 172.31.40.234 dev tunl0 proto bird onlink
Inter-Node Communication
Step 14: Test pod-to-pod communication
Create a second pod on node2:
kubectl run test-pod2 --image=nginx --overrides='{"spec": {"nodeName": "node2"}}'
kubectl get pod test-pod2 -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP               NODE    NOMINATED NODE   READINESS GATES
test-pod2   1/1     Running   0          74s   192.168.104.14   node2   <none>           <none>
Test connectivity using a debug pod with networking tools:
kubectl run debug-pod --image=busybox --rm -it --restart=Never -- ping -c 2 192.168.104.14
64 bytes from 192.168.104.14: seq=1 ttl=62 time=0.622 ms

--- 192.168.104.14 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.622/0.635/0.648 ms
Step 15: Verify encapsulation interface
Check the IPIP tunnel interface:
# On node1
ip link show tunl0
3: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 8981 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
Step 16: Verify IP pool configuration
kubectl get ippools.crd.projectcalico.org -o jsonpath='{.items[0].spec}'
{"allowedUses":["Workload","Tunnel"],"blockSize":26,"cidr":"192.168.0.0/16","ipipMode":"Always","natOutgoing":true,"nodeSelector":"all()","vxlanMode":"Never"}
Key settings:
ipipMode: "Always": Always use IPIP encapsulation
natOutgoing: true: Enable NAT for external traffic
vxlanMode: "Never": Don't use VXLAN
Troubleshooting Commands
Verify Calico components
Verify BlockAffinities
kubectl get blockaffinities.crd.projectcalico.org
Key Takeaways
IP Allocation:
Handled by calico-ipam plugin
Communicates directly with Kubernetes API
Does NOT involve calico-node
Network Configuration:
CNI plugin configures pod networking
Creates veth pairs and local routes
Inter-Node Communication:
BGP distributes routes between nodes
IPIP encapsulation handles inter-node traffic
Routes point to node IPs as next hops
Component Roles:
calico-node: Handles BGP, routes, and Felix
calico-kube-controllers: Manages IPAM cleanup and policies
CNI plugins: Handle pod networking setup
This lab demonstrates the complete flow of Calico CNI operations in Kubernetes, from pod creation to inter-node communication.

