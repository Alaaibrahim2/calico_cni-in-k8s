Calico CNI Deep Dive – Complete Single-Block Lab

Architecture Overview
Calico consists of:
- calico-node (DaemonSet on every node): BGP, routing, Felix
- calico-kube-controllers (single Deployment): IPAM cleanup, policy sync
- CNI plugins:
  /opt/cni/bin/calico
  /opt/cni/bin/calico-ipam
- CNI config:
  /etc/cni/net.d/10-calico.conflist

==================================================

Verify Calico Components
Command:
kubectl get pods -n kube-system | grep calico

Output:
calico-kube-controllers-658d97c59c-zfp99   1/1 Running   45   87d
calico-node-4c5z5                          1/1 Running   37   87d
calico-node-kqfgr                          1/1 Running   37   87d
calico-node-nn766                          1/1 Running   37   87d

==================================================

Pod Creation Flow

Step 1: Scheduler assigns Pod, kubelet creates it
Command:
kubectl run test-pod --image=nginx --overrides='{"spec":{"nodeName":"node1"}}'

Output:
pod/test-pod created

--------------------------------------------------

Step 2: kubelet communicates with CRI
containerd socket: /var/run/containerd/containerd.sock
CRI-O socket:      /var/run/crio/crio.sock

--------------------------------------------------

Step 3: CRI creates Pod Sandbox
- Network namespace created
- Pause container started
Example namespace path:
/var/run/netns/cni-xxxx

--------------------------------------------------

Step 4: kubelet reads CNI configuration
Command:
cat /etc/cni/net.d/10-calico.conflist

Output:
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

--------------------------------------------------

Step 5: kubelet executes CNI plugin
Binary executed:
/opt/cni/bin/calico

==================================================

IP Allocation (IPAM)

Step 6: calico plugin calls IPAM plugin
Binary:
/opt/cni/bin/calico-ipam

--------------------------------------------------

Step 7: IPAM plugin talks directly to Kubernetes API
API Server:
https://10.96.0.1:443

Actions:
- Reads IPPool configuration
- Allocates IP block: 192.168.166.128/26
- Assigns pod IP: 192.168.166.170
- Creates IPAMBlock
- Creates BlockAffinity

IMPORTANT:
calico-ipam talks directly to Kubernetes API
calico-ipam does NOT talk to calico-node

--------------------------------------------------

Step 8: Verify Pod IP
Command:
kubectl get pod test-pod -o wide

Output:
NAME       READY STATUS  AGE  IP                NODE
test-pod   1/1   Running 27s  192.168.166.170   node1

--------------------------------------------------

Step 9: Verify BlockAffinities
Command:
kubectl get blockaffinities.crd.projectcalico.org

Output:
master-192-168-219-64-26
node1-192-168-166-128-26
node2-192-168-104-0-26

Meaning:
IPAMBlock     = actual IP range
BlockAffinity = node ownership of the block

==================================================

Network Configuration

Step 10: CNI configures pod networking
- Pod IP assigned as /32
- veth pair created
- Local host route added

Command:
ip route show | grep 192.168.166.170

Output:
192.168.166.170 dev cali7fba7a35b74 scope link

--------------------------------------------------

Step 11: calico-node watches Kubernetes API
Watches:
- IPAMBlocks
- BlockAffinities
- NetworkPolicies

Felix programs kernel routes and iptables.

--------------------------------------------------

Step 12: BGP peering
Command:
ss -tn | grep ':179'

Output:
ESTAB 0 0 172.31.34.115:179 172.31.40.234:43559
ESTAB 0 0 172.31.34.115:179 172.31.35.22:56249

Node1 peers with:
- master 172.31.40.234
- node2  172.31.35.22

--------------------------------------------------

Step 13: Route distribution
Command:
ip route show | grep tunl0

Output:
192.168.104.0/26 via 172.31.35.22 dev tunl0 proto bird onlink
192.168.219.64/26 via 172.31.40.234 dev tunl0 proto bird onlink

==================================================

Inter-Node Pod Communication

Step 14: Create pod on node2
Command:
kubectl run test-pod2 --image=nginx --overrides='{"spec":{"nodeName":"node2"}}'

Command:
kubectl get pod test-pod2 -o wide

Output:
NAME        READY STATUS  AGE  IP              NODE
test-pod2   1/1   Running 74s  192.168.104.14  node2

--------------------------------------------------

Test connectivity
Command:
kubectl run debug-pod --image=busybox --rm -it --restart=Never -- ping -c 2 192.168.104.14

Output:
64 bytes from 192.168.104.14: seq=1 ttl=62 time=0.622 ms

2 packets transmitted, 2 received, 0% packet loss

--------------------------------------------------

Step 15: Verify IPIP tunnel
Command:
ip link show tunl0

Output:
tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 8981
link/ipip 0.0.0.0 brd 0.0.0.0

--------------------------------------------------

Step 16: Verify IPPool
Command:
kubectl get ippools.crd.projectcalico.org -o jsonpath='{.items[0].spec}'

Output:
{
  "cidr":"192.168.0.0/16",
  "blockSize":26,
  "ipipMode":"Always",
  "vxlanMode":"Never",
  "natOutgoing":true
}

==================================================

Key Takeaways

- IP allocation is done by calico-ipam
- calico-ipam communicates directly with Kubernetes API
- CNI plugin creates veth pairs and /32 routes
- BGP distributes routes between nodes
- IPIP encapsulation handles inter-node traffic
- calico-node handles BGP, routing, Felix
- calico-kube-controllers handles IPAM cleanup

END OF LAB

