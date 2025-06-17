# HA-KubeNet: Highly Available Kubernetes Network Infrastructure

This project implements a **3-node hyper-converged HA cluster** on Ubuntu 22.04, combining network services with a MicroK8s Kubernetes control plane. It **eliminates single points of failure** by running DHCP, DNS, NFS and applications redundantly across the nodes.  Keepalived (VRRP) is used to manage a shared Virtual IP for core services [oai_citation:0‡digitalocean.com](https://www.digitalocean.com/community/questions/navigating-high-availability-with-keepalived#:~:text=%60Keepalived%60%20is%20an%20open,if%20the%20primary%20server%20fails) [oai_citation:1‡medium.com](https://medium.com/@simardeep.oberoi/ensuring-high-availability-in-kubernetes-a-guide-to-multiple-master-nodes-22be302a4726#:~:text=availability).  The Kubernetes cluster is inherently HA with 3 nodes [oai_citation:2‡microk8s.io](https://microk8s.io/docs/high-availability#:~:text=,with%20three%20or%20more%20nodes), and MetalLB provides a bare-metal LoadBalancer for service IPs.  The network is fully segmented by VLANs with redundant distribution/core routers for gateway failover, and NAT/PAT on the core routers to share a WAN IP. Key features include:

- **No single point of failure:** Core services (DHCP, DNS, NFS) and app workloads run on all 3 nodes; active services fail over using VRRP [oai_citation:3‡medium.com](https://medium.com/@simardeep.oberoi/ensuring-high-availability-in-kubernetes-a-guide-to-multiple-master-nodes-22be302a4726#:~:text=availability) [oai_citation:4‡digitalocean.com](https://www.digitalocean.com/community/questions/navigating-high-availability-with-keepalived#:~:text=,over%20should%20the%20Master%20fail).  
- **High uptime:** If one node fails, Keepalived promotes a backup and Kubernetes reschedules pods elsewhere.  
- **VLAN segmentation:** Five VLANs (Developer, Designer, HR, Marketing, DMZ) isolate traffic. Distribution switches use **`ip helper-address 172.16.16.10`** to relay DHCP requests to the master DHCP server [oai_citation:5‡cisco.com](http://www.cisco.com/en/US/docs/ios/12_4t/ip_addr/configuration/guide/htdhcpre.html#:~:text=,address%20interface%20configuration%20command). Gateway redundancy is provided by VRRP/HSRP on the distribution switches.  
- **Kubernetes networking:** Calico (default CNI) and MetalLB handle pod networking and external service IPs. MetalLB assigns a pool (`172.16.16.100`–`172.16.16.180`) of IPs for LoadBalancer services [oai_citation:6‡redhat.com](https://www.redhat.com/en/blog/deploying-a-high-availability-fault-tolerant-kubernetes-service-on-baremetal-clusters-with-metallb-bgp#:~:text=Working%20with%20bare,through%20an%20external%20IP%20address).  
- **Core routing and NAT:** Two core routers (R1/R2) form port-channels to distribution switches with equal-cost links. They use PAT (`ip nat overload`) to allow 192.168.0.0/16 and 172.16.16.0/24 clients to share the public IP [oai_citation:7‡firewall.cx](https://www.firewall.cx/cisco/cisco-routers/cisco-router-nat-overload.html#:~:text=NAT%20,feature%20of%20TCP%2FUDP%20ports%20translation).  

**IP Addressing Summary:**  
- **Node IPs:** node1=`172.16.16.11`, node2=`172.16.16.12`, node3=`172.16.16.13` (ens33 on each).  
- **Virtual IP:** `172.16.16.10` (managed by Keepalived).  
- **MetalLB Pool:** `172.16.16.100`–`172.16.16.180`.  
- **VLAN Gateways:** VLAN10-40 .254 via VRRP, VLAN50 (DMZ) .254 via VRRP on DisSw1/2.  
- **DMZ network:** 172.16.16.0/24 (gateway .254).  

## Network Topology

*(Placeholder for network topology diagram illustrating the 3-node cluster connected to access/distribution/core layers with VLAN segmentation and redundant links.)*

## Server Layer (Nodes 1–3)

Each node runs Keepalived, ISC DHCP, BIND9, NFS server and is part of the MicroK8s cluster. Keepalived creates a VRRP instance on interface **ens33** with a shared VIP `172.16.16.10`, bringing up DHCP, DNS and NFS only on the MASTER node. Below are the key configurations:

**HA Service Handler Script (`/usr/local/bin/ha_service_handler.sh` on all nodes):**  
```bash
#!/bin/bash
# This script is called by Keepalived to start/stop services based on node state.
SERVICES="isc-dhcp-server bind9 nfs-kernel-server"
LOGFILE="/var/log/keepalived_handler.log"

log_action() {
    echo "$(date): $1" >> $LOGFILE
}

case "$1" in
    master)
        log_action "Transitioning to MASTER. Starting services."
        for S in $SERVICES; do systemctl start $S; done
        ;;
    backup|fault)
        log_action "Transitioning to BACKUP/FAULT. Stopping services."
        for S in $SERVICES; do systemctl stop $S; done
        ;;
    *)
        log_action "Unknown state: $1"; exit 1
        ;;
esac
exit 0
```  
*(Make executable: `sudo chmod +x /usr/local/bin/ha_service_handler.sh`.)*

**Keepalived (Node1, `/etc/keepalived/keepalived.conf`):** Node1 is MASTER (highest priority).  
```conf
global_defs { router_id NODE1 }
vrrp_instance VI_CLUSTER {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication { auth_type PASS; auth_pass admin; }
    virtual_ipaddress { 172.16.16.10/24 dev ens33 }
    notify_master "/usr/local/bin/ha_service_handler.sh master"
    notify_backup "/usr/local/bin/ha_service_handler.sh backup"
    notify_fault  "/usr/local/bin/ha_service_handler.sh fault"
}
```

**Keepalived (Node2, `/etc/keepalived/keepalived.conf`):** Node2 is BACKUP (priority 100).  
```conf
global_defs { router_id NODE2 }
vrrp_instance VI_CLUSTER {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication { auth_type PASS; auth_pass admin; }
    virtual_ipaddress { 172.16.16.10/24 dev ens33 }
    notify_master "/usr/local/bin/ha_service_handler.sh master"
    notify_backup "/usr/local/bin/ha_service_handler.sh backup"
    notify_fault  "/usr/local/bin/ha_service_handler.sh fault"
}
```

**Keepalived (Node3, `/etc/keepalived/keepalived.conf`):** Node3 is BACKUP (priority 50).  
```conf
global_defs { router_id NODE3 }
vrrp_instance VI_CLUSTER {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 50
    advert_int 1
    authentication { auth_type PASS; auth_pass admin; }
    virtual_ipaddress { 172.16.16.10/24 dev ens33 }
    notify_master "/usr/local/bin/ha_service_handler.sh master"
    notify_backup "/usr/local/bin/ha_service_handler.sh backup"
    notify_fault  "/usr/local/bin/ha_service_handler.sh fault"
}
```

**ISC DHCP Server (all nodes):** Bound to `ens33`, providing addresses for each VLAN via relay on switches.  
- `/etc/default/isc-dhcp-server`:  
  ```bash
  INTERFACESv4="ens33"
  ```  
- `/etc/dhcp/dhcpd.conf` (scopes):  
  ```conf
  option domain-name "cluster.local";
  option domain-name-servers 172.16.16.10;
  default-lease-time 7200;
  max-lease-time 14400;
  authoritative;
  log-facility local7;

  subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.50 192.168.10.200;
    option routers 192.168.10.254;
  }
  subnet 192.168.20.0 netmask 255.255.255.0 {
    range 192.168.20.50 192.168.20.200;
    option routers 192.168.20.254;
  }
  subnet 192.168.30.0 netmask 255.255.255.0 {
    range 192.168.30.50 192.168.30.200;
    option routers 192.168.30.254;
  }
  subnet 192.168.40.0 netmask 255.255.255.0 {
    range 192.168.40.50 192.168.40.200;
    option routers 192.168.40.254;
  }
  subnet 172.16.16.0 netmask 255.255.255.0 {
    range 172.16.16.30 172.16.16.99;
    option routers 172.16.16.1;
  }
  ```

**BIND9 DNS Server (all nodes):** Provides name resolution for the cluster and forwards requests to public DNS.  
- `/etc/bind/named.conf.options`:  
  ```conf
  options {
      directory "/var/cache/bind";
      listen-on { any; };
      allow-query { any; };
      recursion yes;
      forwarders { 8.8.8.8; 1.1.1.1; };
      dnssec-validation auto;
  };
  ```

**NFS Kernel Server (all nodes):** Exports shared storage for Kubernetes PVs and data. Create directories with `mkdir -p /srv/nfs/data /srv/nfs/kubernetes` and permissions `chown nobody:nogroup`, `chmod 777` on them.  
- `/etc/exports`:  
  ```conf
  /srv/nfs/data        192.168.0.0/16(rw,sync,no_subtree_check) 172.16.16.0/24(rw,sync,no_subtree_check)
  /srv/nfs/kubernetes  172.16.16.0/24(rw,sync,no_subtree_check,no_root_squash)
  ```

**MicroK8s Cluster:** Installed on all nodes with one as initial master (node1). After joining node2/node3, enable HA and necessary addons:  
```bash
# On node1 (initial)
snap install microk8s --classic --channel=1.21/stable
microk8s add-node      # run and join tokens on node2/node3

# On node2/node3 (joining):
snap install microk8s --classic --channel=1.21/stable
microk8s join <IP>:<port>/<token>

# Enable services on any node:
microk8s enable dns dashboard storage ingress
microk8s enable metallb
```

**MetalLB Configuration:** After enabling, apply a ConfigMap (`metallb-config.yaml`) to set the L2 address pool:  
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.16.16.100-172.16.16.180
```
*(Apply with `microk8s kubectl apply -f metallb-config.yaml`.)*

**NFS Persistent Volume Provisioner:** We deploy an external NFS client provisioner so Kubernetes can create PVs on our NFS share. Example manifests (`rbac.yaml`, `class.yaml`, `deployment.yaml`):  

- `rbac.yaml` (ServiceAccount, Roles/Bindings for the provisioner):  
  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: nfs-client-provisioner
    namespace: default
  ---
  kind: ClusterRole
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: nfs-client-provisioner-runner
  rules:
    - apiGroups: [""]
      resources: ["nodes", "persistentvolumes", "persistentvolumeclaims"]
      verbs: ["get","list","watch","create","delete","update"]
    - apiGroups: ["storage.k8s.io"]
      resources: ["storageclasses"]
      verbs: ["get","list","watch"]
    - apiGroups: [""]
      resources: ["events"]
      verbs: ["create","update","patch"]
  ---
  kind: ClusterRoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: run-nfs-client-provisioner
  subjects:
    - kind: ServiceAccount
      name: nfs-client-provisioner
      namespace: default
  roleRef:
    kind: ClusterRole
    name: nfs-client-provisioner-runner
    apiGroup: rbac.authorization.k8s.io
  ---
  kind: Role
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: leader-locking-nfs-client-provisioner
    namespace: default
  rules:
    - apiGroups: [""]
      resources: ["endpoints"]
      verbs: ["get","list","watch","create","update","patch"]
  ---
  kind: RoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: leader-locking-nfs-client-provisioner
    namespace: default
  subjects:
    - kind: ServiceAccount
      name: nfs-client-provisioner
      namespace: default
  roleRef:
    kind: Role
    name: leader-locking-nfs-client-provisioner
    apiGroup: rbac.authorization.k8s.io
  ```

- `class.yaml` (StorageClass for the NFS provisioner):  
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: nfs-client
  provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
  parameters:
    archiveOnDelete: "false"
  ```

- `deployment.yaml` (NFS provisioner Deployment):  
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nfs-client-provisioner
    labels: { app: nfs-client-provisioner }
    namespace: default
  spec:
    replicas: 1
    selector:
      matchLabels: { app: nfs-client-provisioner }
    template:
      metadata:
        labels: { app: nfs-client-provisioner }
      spec:
        serviceAccountName: nfs-client-provisioner
        containers:
          - name: nfs-client-provisioner
            image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
            volumeMounts:
              - name: nfs-client-root
                mountPath: /persistentvolumes
            env:
              - name: PROVISIONER_NAME
                value: k8s-sigs.io/nfs-subdir-external-provisioner
              - name: NFS_SERVER
                value: 172.16.16.10
              - name: NFS_PATH
                value: /srv/nfs/kubernetes
        volumes:
          - name: nfs-client-root
            nfs:
              server: 172.16.16.10
              path: /srv/nfs/kubernetes
  ```

**Stateful Application:** The cluster runs a simple todo-app with a PostgreSQL backend. Example manifests:  

- `postgres-secret.yaml` (contains `POSTGRES_PASSWORD: YWRtaW4=` which is “admin” in base64):  
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: postgres-secret
  type: Opaque
  data:
    POSTGRES_PASSWORD: YWRtaW4=
  ```
- `postgres-statefulset.yaml` (Service + StatefulSet for PostgreSQL):  
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: postgres-svc
    labels: { app: postgres }
  spec:
    ports:
      - port: 5432
    clusterIP: None
    selector: { app: postgres }
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: postgres-statefulset
  spec:
    serviceName: "postgres-svc"
    replicas: 1
    selector:
      matchLabels: { app: postgres }
    template:
      metadata:
        labels: { app: postgres }
      spec:
        securityContext: { fsGroup: 999 }
        containers:
          - name: postgres
            image: postgres:13
            ports:
              - containerPort: 5432
            env:
              - name: POSTGRES_USER
                value: todo_user
              - name: POSTGRES_DB
                value: todo_db
              - name: POSTGRES_PASSWORD
                valueFrom:
                  secretKeyRef: { name: postgres-secret, key: POSTGRES_PASSWORD }
              - name: PGDATA
                value: /var/lib/postgresql/data/pgdata
            volumeMounts:
              - name: postgres-persistent-storage
                mountPath: /var/lib/postgresql/data
                subPath: pgdata
    volumeClaimTemplates:
      - metadata:
          name: postgres-persistent-storage
        spec:
          accessModes: ["ReadWriteOnce"]
          storageClassName: nfs-client
          resources:
            requests: { storage: 1Gi }
  ```
- `nodejs-app.yaml` (Deployment + Service for the web front-end):  
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: todo-app-deployment
  spec:
    replicas: 2
    selector:
      matchLabels: { app: todo-app }
    template:
      metadata:
        labels: { app: todo-app }
      spec:
        containers:
          - name: todo-app
            image: kodekloud/webapp-color
            ports:
              - containerPort: 8080
            env:
              - name: DB_HOST
                value: "postgres-svc"
              - name: DB_USER
                value: "todo_user"
              - name: DB_DATABASE
                value: "todo_db"
              - name: DB_PORT
                value: "5432"
              - name: DB_PASSWORD
                valueFrom:
                  secretKeyRef: { name: postgres-secret, key: POSTGRES_PASSWORD }
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: todo-app-service
  spec:
    type: LoadBalancer
    selector:
      app: todo-app
    ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
  ```
The frontend service will receive an external IP (e.g. `172.16.16.100`) from MetalLB’s pool.  

## Core Layer (Routers R1, R2)

These routers form the L3 core with redundant links to distribution switches (port-channel 1/2) and a WAN uplink (FastEthernet0/0 via DHCP). They use NAT overload (PAT) to allow all internal subnets to share the internet IP [oai_citation:8‡firewall.cx](https://www.firewall.cx/cisco/cisco-routers/cisco-router-nat-overload.html#:~:text=NAT%20,feature%20of%20TCP%2FUDP%20ports%20translation). Key configurations:

**R1:**  
``` 
interface Port-channel1
  no shutdown
  ip address 10.0.0.2 255.255.255.252
  hold-queue 150 in
interface Port-channel2
  no shutdown
  ip address 10.0.0.14 255.255.255.252
  hold-queue 150 in
interface FastEthernet0/0
  no shutdown
  ip address dhcp
interface FastEthernet2/0
  no shutdown
  channel-group 2
interface FastEthernet3/0
  no shutdown
  channel-group 2
interface FastEthernet4/0
  no shutdown
  channel-group 1
interface FastEthernet5/0
  no shutdown
  channel-group 1

ip route 192.168.0.0 255.255.0.0 10.0.0.1
ip route 192.168.0.0 255.255.0.0 10.0.0.13 10

interface Port-channel1
  ip nat inside
interface Port-channel2
  ip nat inside
interface FastEthernet0/0
  ip nat outside
ip nat inside source list 1 interface FastEthernet0/0 overload
access-list 1 permit 192.168.0.0 0.0.255.255
access-list 1 permit 172.16.16.0 0.0.0.255
```
This marks internal links (`ip nat inside`) and the WAN link (`ip nat outside`), then enables NAT overload (PAT) for the 192.168.0.0/16 and 172.16.16.0/24 networks [oai_citation:9‡firewall.cx](https://www.firewall.cx/cisco/cisco-routers/cisco-router-nat-overload.html#:~:text=NAT%20,feature%20of%20TCP%2FUDP%20ports%20translation).

**R2:**  
``` 
interface Port-channel1
  no shutdown
  ip address 10.0.0.6 255.255.255.252
  hold-queue 150 in
interface Port-channel2
  no shutdown
  ip address 10.0.0.10 255.255.255.252
  hold-queue 150 in
interface FastEthernet0/0
  no shutdown
  ip address dhcp
interface FastEthernet1/0
  no shutdown
  channel-group 2
interface FastEthernet2/0
  no shutdown
  channel-group 2
interface FastEthernet3/0
  no shutdown
  channel-group 1
interface FastEthernet4/0
  no shutdown
  channel-group 1

ip route 192.168.0.0 255.255.0.0 10.0.0.9
ip route 192.168.0.0 255.255.0.0 10.0.0.5 10

interface Port-channel1
  ip nat inside
interface Port-channel2
  ip nat inside
interface FastEthernet0/0
  ip nat outside
ip nat inside source list 1 interface FastEthernet0/0 overload
access-list 1 permit 192.168.0.0 0.0.255.255
access-list 1 permit 172.16.16.0 0.0.0.255
```

**Core VPN/NAT Notes:** The core routers’ NAT configuration implements PAT so that multiple hosts appear as one public IP [oai_citation:10‡firewall.cx](https://www.firewall.cx/cisco/cisco-routers/cisco-router-nat-overload.html#:~:text=NAT%20,feature%20of%20TCP%2FUDP%20ports%20translation). Outbound traffic from VLAN subnets will be translated on R1/R2’s outside interface. 

## Distribution Layer (DisSw01, DisSw02)

These multilayer switches handle inter-VLAN routing and gateway redundancy. Each has SVIs (virtual interfaces) for VLANs 10,20,30,40,50. We use HSRP/VRRP (`standby`) for gateways:

- **VLANs:** 10 (Dev),20 (Design),30 (HR),40 (Marketing),50 (DMZ).
- **Subnet gateways:** .254 in each VLAN. Distribution switches use `standby <VLAN> ip 192.168.X.254` with priorities (DisSw01 priority 110 on VLAN10/20 for master, etc.) to form a virtual gateway. Master is active, backup(s) wait to preempt [oai_citation:11‡digitalocean.com](https://www.digitalocean.com/community/questions/navigating-high-availability-with-keepalived#:~:text=,over%20should%20the%20Master%20fail).
- **DHCP relay:** Each SVI has `ip helper-address 172.16.16.10` pointing to the DHCP VIP (Keepalived) [oai_citation:12‡cisco.com](http://www.cisco.com/en/US/docs/ios/12_4t/ip_addr/configuration/guide/htdhcpre.html#:~:text=,address%20interface%20configuration%20command).
- **Routing:** DisSw01/02 have static default routes to R1/R2 via port-channels (shown below).

**DisSw01 Configuration:**  
``` 
vlan 10 name Developer
vlan 20 name Designer
vlan 30 name HR
vlan 40 name Marketing
vlan 50 name DMZ_ZONE

interface Port-channel1
  switchport
  switchport trunk encap dot1q
  switchport trunk allowed vlan 1,10,20,30,40,50
  no shutdown
interface Port-channel2
  no switchport
  channel-group 2 mode on
interface Port-channel3
  no switchport
  channel-group 3 mode on
interface Ethernet2/0
  switchport trunk encap dot1q
  switchport trunk allowed vlan 1,10,20,30,40,50
  no shutdown

interface Vlan10
  no shutdown
  ip address 192.168.10.253 255.255.255.0
  ip helper-address 172.16.16.10
  standby 10 ip 192.168.10.254
  standby 10 priority 110
  standby 10 preempt
interface Vlan20
  no shutdown
  ip address 192.168.20.253 255.255.255.0
  ip helper-address 172.16.16.10
  standby 20 ip 192.168.20.254
  standby 20 priority 110
  standby 20 preempt
interface Vlan30
  no shutdown
  ip address 192.168.30.252 255.255.255.0
  ip helper-address 172.16.16.10
  standby 30 ip 192.168.30.254
  standby 30 priority 90
  standby 30 preempt
interface Vlan40
  no shutdown
  ip address 192.168.40.252 255.255.255.0
  ip helper-address 172.16.16.10
  standby 40 ip 192.168.40.254
  standby 40 priority 90
  standby 40 preempt
interface Vlan50
  no shutdown
  ip address 172.16.16.252 255.255.255.0
  ip helper-address 172.16.16.10
  standby 50 ip 172.16.16.254
  standby 50 priority 90
  standby 50 preempt

spanning-tree vlan 10,20 root primary
spanning-tree vlan 30,40,50 root secondary
ip domain-name admin.com
crypto key generate rsa modulus 1024
username admin secret admin
line vty 0 4
 password admin
 login local
 transport input ssh
```

**DisSw02 Configuration:**  
``` 
vlan 10 name Developer
vlan 20 name Designer
vlan 30 name HR
vlan 40 name Marketing
vlan 50 name DMZ_ZONE

interface Port-channel1
  switchport
  switchport trunk encap dot1q
  switchport trunk allowed vlan 1,10,20,30,40,50
  no shutdown
interface Port-channel2
  no switchport
  channel-group 2 mode on
interface Port-channel3
  no switchport
  channel-group 3 mode on
interface Ethernet2/0
  switchport trunk encap dot1q
  switchport trunk allowed vlan 1,10,20,30,40,50
  no shutdown

interface Vlan10
  no shutdown
  ip address 192.168.10.252 255.255.255.0
  ip helper-address 172.16.16.10
  standby 10 ip 192.168.10.254
  standby 10 priority 90
  standby 10 preempt
interface Vlan20
  no shutdown
  ip address 192.168.20.252 255.255.255.0
  ip helper-address 172.16.16.10
  standby 20 ip 192.168.20.254
  standby 20 priority 90
  standby 20 preempt
interface Vlan30
  no shutdown
  ip address 192.168.30.253 255.255.255.0
  ip helper-address 172.16.16.10
  standby 30 ip 192.168.30.254
  standby 30 priority 110
  standby 30 preempt
interface Vlan40
  no shutdown
  ip address 192.168.40.253 255.255.255.0
  ip helper-address 172.16.16.10
  standby 40 ip 192.168.40.254
  standby 40 priority 110
  standby 40 preempt
interface Vlan50
  no shutdown
  ip address 172.16.16.253 255.255.255.0
  ip helper-address 172.16.16.10
  standby 50 ip 172.16.16.254
  standby 50 priority 110
  standby 50 preempt

spanning-tree vlan 30,40,50 root primary
spanning-tree vlan 10,20 root secondary
ip domain-name admin.com
crypto key generate rsa modulus 1024
username admin secret admin
line vty 0 4
 password admin
 login local
 transport input ssh
```

**Default Routes:** Both distribution switches point to R1/R2 for Internet access:  
``` 
DisSw01> ip route 0.0.0.0/0 10.0.0.2
DisSw01> ip route 0.0.0.0/0 10.0.0.6 distance 10

DisSw02> ip route 0.0.0.0/0 10.0.0.10
DisSw02> ip route 0.0.0.0/0 10.0.0.14 distance 10
```
These provide active/passive or load-shared uplinks via R1 (10.0.0.2/10.0.0.6) and R2 (10.0.0.10/10.0.0.14).

## Access Layer (AcSw01, AcSw02, AcSw03)

Three access switches connect end devices and uplink to the distribution. All have VLANs 10–50 defined and trunk ports to uplinks. Access ports are assigned to specific VLANs as needed:

- **VLANs 10–50:** Each switch defines VLAN 10 (Developer), 20 (Designer), 30 (HR), 40 (Marketing), 50 (DMZ).  
- **Trunks:** Links to distribution (Eth0/0–0/1 or Eth1/0–1/1) are configured as dot1q trunks carrying VLANs 1,10,20,30,40,50.  
- **Access Ports:** End-host ports (e.g. Eth0/2–0/3, Eth1/0–1/1) are set to `switchport mode access` and assigned to the appropriate VLAN. Portfast is enabled for quick connectivity.  
- **No IP Routing:** These are layer-2 switches (no SVI IPs on access layer).  

**AcSw01:**  
``` 
hostname AcSw01
vlan 10 name Developer
vlan 20 name Designer
vlan 30 name HR
vlan 40 name Marketing
vlan 50 name DMZ_ZONE

interface eth0/0
  switchport trunk encap dot1q
  switchport mode trunk
  switchport nonegotiate
interface eth0/1
  switchport trunk encap dot1q
  switchport mode trunk
  switchport nonegotiate

interface eth1/0
  switchport mode access
  switchport access vlan 10
  spanning-tree portfast
interface eth1/1
  switchport mode access
  switchport access vlan 10
  spanning-tree portfast

interface eth0/2
  switchport mode access
  switchport access vlan 20
  spanning-tree portfast
interface eth0/3
  switchport mode access
  switchport access vlan 20
  spanning-tree portfast
```

**AcSw02:**  
``` 
hostname AcSw02
vlan 10 name Developer
vlan 20 name Designer
vlan 30 name HR
vlan 40 name Marketing
vlan 50 name DMZ_ZONE

interface eth0/0
  switchport trunk encap dot1q
  switchport mode trunk
  switchport nonegotiate
interface eth1/1
  switchport trunk encap dot1q
  switchport mode trunk
  switchport nonegotiate

interface eth1/0
  switchport mode access
  switchport access vlan 40
  spanning-tree portfast
interface eth0/3
  switchport mode access
  switchport access vlan 40
  spanning-tree portfast

interface eth0/2
  switchport mode access
  switchport access vlan 30
  spanning-tree portfast
interface eth0/1
  switchport mode access
  switchport access vlan 30
  spanning-tree portfast
```

**AcSw03:**  
``` 
hostname AcSw03
vlan 10 name Developer
vlan 20 name Designer
vlan 30 name HR
vlan 40 name Marketing
vlan 50 name DMZ_ZONE

interface eth0/0
  switchport trunk encap dot1q
  switchport mode trunk
  switchport nonegotiate
  switchport trunk allowed vlan 1,10,20,30,40,50
interface eth0/1
  switchport trunk encap dot1q
  switchport mode trunk
  switchport nonegotiate
  switchport trunk allowed vlan 1,10,20,30,40,50

interface eth0/2
  switchport mode access
  switchport access vlan 50
  spanning-tree portfast edge
interface eth0/3
  switchport mode access
  switchport access vlan 50
  spanning-tree portfast edge

interface eth1/0
  switchport mode access
  switchport access vlan 50
  spanning-tree portfast edge
```

Each access switch is simply managing VLAN ports; no routing or IP is set on these devices. End-user machines on each VLAN will obtain an IP via the DHCP relay through `172.16.16.10` [oai_citation:13‡cisco.com](http://www.cisco.com/en/US/docs/ios/12_4t/ip_addr/configuration/guide/htdhcpre.html#:~:text=,address%20interface%20configuration%20command).

## Final Notes

After deploying the above configuration:

- The **web app** can be reached at the MetalLB-assigned IP (e.g., http://172.16.16.100) from any network.
- The **HA design** ensures that any single node or switch/router failure will not interrupt services. Keepalived VRRP/HSRP will fail over the VIP and gateway, and Kubernetes will reschedule pods on the remaining nodes, maintaining continuous availability [oai_citation:14‡medium.com](https://medium.com/@simardeep.oberoi/ensuring-high-availability-in-kubernetes-a-guide-to-multiple-master-nodes-22be302a4726#:~:text=Conclusion) [oai_citation:15‡digitalocean.com](https://www.digitalocean.com/community/questions/navigating-high-availability-with-keepalived#:~:text=,over%20should%20the%20Master%20fail).

**Sources:** Design principles and protocol usage are based on industry best practices (e.g., VRRP for VIPs [oai_citation:16‡digitalocean.com](https://www.digitalocean.com/community/questions/navigating-high-availability-with-keepalived#:~:text=%60Keepalived%60%20is%20an%20open,if%20the%20primary%20server%20fails) [oai_citation:17‡digitalocean.com](https://www.digitalocean.com/community/questions/navigating-high-availability-with-keepalived#:~:text=,over%20should%20the%20Master%20fail), DHCP relay via `ip helper` [oai_citation:18‡cisco.com](http://www.cisco.com/en/US/docs/ios/12_4t/ip_addr/configuration/guide/htdhcpre.html#:~:text=,address%20interface%20configuration%20command), NAT overload for PAT [oai_citation:19‡firewall.cx](https://www.firewall.cx/cisco/cisco-routers/cisco-router-nat-overload.html#:~:text=NAT%20,feature%20of%20TCP%2FUDP%20ports%20translation), and Kubernetes HA requirements [oai_citation:20‡microk8s.io](https://microk8s.io/docs/high-availability#:~:text=,with%20three%20or%20more%20nodes) [oai_citation:21‡medium.com](https://medium.com/@simardeep.oberoi/ensuring-high-availability-in-kubernetes-a-guide-to-multiple-master-nodes-22be302a4726#:~:text=availability)). All configuration commands above are taken from the provided templates.
