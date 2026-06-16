# Architecture & How It Works

This document explains the *why* behind the design — useful both as study notes and to talk through in an interview.

## The big picture

```
                         Internet  (NSG opens ONLY 22, 80, 443)
                                       │
                  DNS A record         │
                  k8s.chishty.me ──────┴────►  worker public IP  :80 / :443
                                       │
   ┌───────────────────────────────────┴───────────────────────────────────┐
   │                        Azure VNet  10.20.0.0/16                          │
   │                                                                         │
   │   ┌──────────────── k8s-cp (10.20.1.4) ────────────────┐                │
   │   │  CONTROL PLANE (static pods, made by kubeadm)        │               │
   │   │   • kube-apiserver  ← the only thing kubectl talks to│               │
   │   │   • etcd            ← key/value store of cluster state│              │
   │   │   • kube-scheduler  ← decides which node runs a Pod  │               │
   │   │   • controller-mgr  ← reconciles desired vs actual   │               │
   │   │   • calico + kube-proxy                              │               │
   │   └──────────────────────────────────────────────────────┘             │
   │                                                                         │
   │   ┌──────────────── k8s-w1 (10.20.1.5) ─────────────────┐               │
   │   │  WORKER                                              │               │
   │   │   • ingress-nginx  (hostNetwork → binds :80/:443)    │               │
   │   │   • web Deployment: 3× Pod (nginxdemos/hello)        │               │
   │   │   • cert-manager solver pods (during ACME challenge) │               │
   │   │   • kubelet + containerd + calico + kube-proxy       │               │
   │   └──────────────────────────────────────────────────────┘             │
   └───────────────────────────────────────────────────────────────────────┘
```

## The request path (what happens when someone visits the site)

1. **DNS** — `k8s.chishty.me` resolves (A record) to the **worker's public IP**.
2. **NSG** — the packet hits Azure's firewall; only 22/80/443 are allowed in, so 443 passes.
3. **ingress-nginx** — runs with `hostNetwork: true`, so it's listening directly on the worker's `:443`. It **terminates TLS** using the certificate cert-manager stored in the `web-tls` Secret.
4. **Ingress rule** — matches host `k8s.chishty.me`, path `/`, and forwards to the `web` **Service**.
5. **Service (ClusterIP)** — load-balances across the healthy `web` Pods (selected by the label `app: web`) via kube-proxy.
6. **Pod** — one of the three `nginxdemos/hello` pods serves the response (it prints its own name, so you can *see* the load balancing rotate).

## Key design decisions (and the reasoning)

### Why `hostNetwork` for the ingress?
A self-managed cluster has **no cloud load balancer**, so a `Service type=LoadBalancer` would never get an external IP. And only 80/443 are open. Running ingress-nginx as a **DaemonSet with `hostNetwork: true`** makes it bind the node's real ports 80/443, so internet traffic reaches it directly. (On AKS/EKS you'd instead use `type=LoadBalancer` and the cloud provisions an LB.)

### Why Calico in VXLAN mode (not the default IP-in-IP)?
Calico's default manifest uses **IP-in-IP** encapsulation for cross-node pod traffic. **Azure's network silently drops IP-in-IP between VMs**, which black-holes all pod-to-pod traffic that crosses nodes — including DNS queries to CoreDNS. Switching the IP pool to **VXLAN** (a UDP-based overlay the cloud does forward) fixes it. This is *the* classic "kubeadm on a cloud" gotcha.

### Why the control plane runs no app pods
`kubeadm` puts a `node-role.kubernetes.io/control-plane:NoSchedule` **taint** on the control-plane node, so ordinary workloads stay off it and land on the worker. The control plane is reserved for running the cluster itself.

### How TLS is automatic
`cert-manager` watches Ingresses carrying the `cert-manager.io/cluster-issuer` annotation. It requests a cert from **Let's Encrypt** over ACME, proves domain ownership with an **HTTP-01 challenge** (Let's Encrypt fetches a token over port 80 — which is why 80 is open), stores the issued cert in the `web-tls` Secret, and **auto-renews** before expiry. Zero manual cert handling.

## Networking ports (why node-to-node still works behind a 3-port firewall)
The NSG only opens 22/80/443 **to the internet**. The cluster's internal ports — `6443` (API), `10250` (kubelet), `2379-2380` (etcd), and the VXLAN overlay (`4789/udp`) — work because all VMs share the VNet and Azure's default `AllowVnetInBound` rule permits intra-VNet traffic. Least privilege at the edge, open inside.

## Azure → AWS mapping

| This project (Azure) | AWS equivalent |
|----------------------|----------------|
| Virtual Machine (`Standard_B2s`) | EC2 instance |
| VNet / Subnet | VPC / Subnet |
| Network Security Group | Security Group |
| kubeadm on VMs | EKS with self-managed nodes |
| Azure DNS / external registrar | Route 53 |
| (would-be) Azure Key Vault | Secrets Manager |
| Azure Load Balancer (on AKS) | ELB / NLB |

The Kubernetes layer — Pods, Deployments, Services, Ingress, cert-manager, Calico — is **identical** on any cloud. That portability is exactly why learning it at the `kubeadm` level is worth it.
