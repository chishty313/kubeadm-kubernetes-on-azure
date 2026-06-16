# Kubernetes by Hand — A Teaching Walkthrough

> **How to use this file:** Don't run scripts. Read each command, understand the story, then **type it yourself** by hand. The act of typing builds the memory; the story builds the understanding. Every command below is on its own line with a *what / why* so you never type something you don't understand.
>
> Symbols I use:
> - 🖥️ **WHERE** — which machine to type this on (your laptop, the control-plane, or the worker)
> - 📖 **STORY** — the concept and the reason behind the command
> - ✅ **CHECK** — what you should see before moving on

---

## The story so far (read this once before you start)

Imagine you're opening a small delivery company. You need a **dispatch office** that takes orders, keeps the master list of "what trucks should be doing," and assigns work. And you need **trucks** that actually carry the parcels. If a truck breaks down, the office notices and sends another. If orders spike, the office puts more trucks on the road.

That is Kubernetes.

- The **dispatch office** = the **control plane** (the `k8s-cp` machine). Inside it live four clerks: the **API server** (the front desk everyone talks to), **etcd** (the filing cabinet holding the master list), the **scheduler** (decides which truck gets which parcel), and the **controller-manager** (the supervisor who constantly compares "what should be" against "what is" and fixes the difference).
- The **trucks** = the **worker nodes** (`k8s-w1`). Each worker runs a **kubelet** (the driver who takes orders from the office and reports back) and a **container runtime** (the engine — `containerd` — that actually runs your app).
- Your app runs inside **Pods** (a parcel). You never tell a specific truck "carry this." You tell the office "I always want 3 of these parcels in transit," and the office makes it true — forever. That promise is called **desired state**, and the office's endless habit of making reality match it is called **reconciliation**. It's the single most important idea in Kubernetes.

We're going to build this whole company from empty Azure VMs, by hand. By the end you'll have a real cluster serving your app at **https://k8s.chishty.me** with a genuine TLS certificate — and you'll know what every piece is for.

---

# PART A — Build the two machines (on your laptop)

🖥️ **WHERE:** your own laptop terminal, where the Azure CLI is logged in (`az login` already done).

📖 **STORY:** Before Kubernetes can exist, it needs computers to live on. We'll rent two small Linux servers from Azure: one to be the dispatch office (control-plane), one to be a truck (worker). We also build a private street for them (a virtual network) and a security gate (an NSG) that only opens three doors to the public internet: SSH (22) so you can log in, and HTTP (80) + HTTPS (443) so visitors can reach your app. Everything else stays shut.

### A1 — Name everything once, so the later commands read cleanly

```bash
RG=rg-k8s-lab
LOC=southeastasia
VNET=vnet-k8s
SUBNET=snet-k8s
NSG=nsg-k8s
```

📖 **STORY:** These are just shell variables — nicknames. Typing `$RG` later is shorter and prevents typos. They vanish when you close the terminal, which is fine; we use them only during setup. `southeastasia` is the Azure region closest to Bangladesh, so your latency is low.

### A2 — Create the resource group (the folder)

```bash
az group create -n $RG -l $LOC
```

📖 **STORY:** A **resource group** is a folder that holds related cloud things. Its superpower is cleanup: when you're done tonight, deleting this one folder deletes the VMs, the network, everything — and the billing stops. That's your safety net.

### A3 — Create the private network and a subnet inside it

```bash
az network vnet create -g $RG -n $VNET \
  --address-prefix 10.20.0.0/16 \
  --subnet-name $SUBNET --subnet-prefix 10.20.1.0/24
```

📖 **STORY:** A **virtual network (VNet)** is a private street that only your machines live on. `10.20.0.0/16` is the range of private house-numbers (IP addresses) available on that street. The **subnet** `10.20.1.0/24` is one block on that street where we'll put the VMs. Why does this matter for Kubernetes? Because the two machines must be able to talk to each other freely on *many* ports (the office and trucks gossip constantly), and Azure automatically allows all traffic *within* a VNet. So putting both VMs in the same subnet is what lets the cluster form, even though we lock down the public side.

### A4 — Create the security gate (NSG) and open exactly three doors

```bash
az network nsg create -g $RG -n $NSG
```

📖 **STORY:** An **NSG (Network Security Group)** is a firewall — a list of allow/deny rules. Right now it's empty, which (with Azure defaults) means "deny inbound from the internet." We'll punch exactly three holes.

```bash
az network nsg rule create -g $RG --nsg-name $NSG -n allow-ssh \
  --priority 100 --destination-port-ranges 22  --access Allow --protocol Tcp --direction Inbound
```

📖 **STORY:** Door #1: **port 22 (SSH)** so you can log into the servers. `priority 100` — lower numbers are evaluated first; it's just ordering. `direction Inbound` — this rule is about traffic coming *toward* the servers.

```bash
az network nsg rule create -g $RG --nsg-name $NSG -n allow-http \
  --priority 110 --destination-port-ranges 80  --access Allow --protocol Tcp --direction Inbound
```

📖 **STORY:** Door #2: **port 80 (HTTP)**. Two reasons we need it: visitors first arrive on 80 (we'll redirect them to HTTPS), and — crucially — Let's Encrypt will knock on port 80 later to prove you own your domain before it hands you a certificate.

```bash
az network nsg rule create -g $RG --nsg-name $NSG -n allow-https \
  --priority 120 --destination-port-ranges 443 --access Allow --protocol Tcp --direction Inbound
```

📖 **STORY:** Door #3: **port 443 (HTTPS)** — the encrypted front door your real users will use.

```bash
az network vnet subnet update -g $RG --vnet-name $VNET -n $SUBNET --network-security-group $NSG
```

📖 **STORY:** This bolts the gate onto the whole block (subnet), so every VM we place there inherits the same three-doors-only rule. One gate, applied once, protects all machines. This is exactly the "least privilege" story you'll tell in the interview.

### A5 — Create the control-plane VM

```bash
az vm create -g $RG -n k8s-cp \
  --image Ubuntu2204 --size Standard_B2s \
  --vnet-name $VNET --subnet $SUBNET --nsg "" \
  --admin-username azureuser --generate-ssh-keys
```

📖 **STORY:** This rents the dispatch-office machine. `Ubuntu2204` is the OS. `Standard_B2s` gives 2 CPUs / 4 GB — and the control plane *requires* at least 2 CPUs, so don't go smaller here. `--nsg ""` is important: it tells Azure **not** to create a separate per-VM firewall that would shadow the subnet one we just configured — we want our single subnet gate to be the only authority. `--generate-ssh-keys` creates a login key pair on your laptop so you can SSH in without a password.

### A6 — Create the worker VM

```bash
az vm create -g $RG -n k8s-w1 \
  --image Ubuntu2204 --size Standard_B2s \
  --vnet-name $VNET --subnet $SUBNET --nsg "" \
  --admin-username azureuser --generate-ssh-keys
```

📖 **STORY:** Same machine, this one's a truck. It joins the same subnet, so it shares the private street with the office and inherits the same three-door gate.

### A7 — Find the addresses you'll need

```bash
az vm list-ip-addresses -g $RG -o table
```

📖 **STORY:** Every VM has two addresses: a **public IP** (reachable from the internet — you SSH to it, and your domain will point at it) and a **private IP** (only reachable on the VNet street — the cluster uses these to talk internally). This shows the public ones.

```bash
az vm list-ip-addresses -g $RG \
  --query "[].{name:virtualMachine.name, private:virtualMachine.network.privateIpAddresses[0]}" -o table
```

📖 **STORY:** The `--query` part is **JMESPath**, a little filter language that pulls just the fields you want out of Azure's big JSON reply. Here it prints each VM's name and private IP in a tidy table.

**Write these four values on paper** — you'll retype them often:
- `CP_PUBLIC` and `CP_PRIVATE` (the office)
- `W1_PUBLIC` and `W1_PRIVATE` (the truck)

✅ **CHECK:** You can log into both machines:
```bash
ssh azureuser@<CP_PUBLIC>
```
```bash
ssh azureuser@<W1_PUBLIC>
```
If you get a shell prompt on each, Part A is done. Keep both terminals open — call them **CP terminal** and **W1 terminal**.

---

# PART B — Prepare each machine to run Kubernetes (by hand)

📖 **STORY:** A fresh Ubuntu box can't run Kubernetes yet. Think of it like prepping a kitchen before the chefs arrive: turn off the thing that confuses them (swap), install the stove (the container runtime), open the right vents (kernel networking settings), and hire the staff (the `kubeadm`/`kubelet`/`kubectl` tools). You must do **all of Part B on BOTH machines** — first in the CP terminal, then repeat the identical commands in the W1 terminal. Doing it twice by hand is *good* — the second time it'll already feel familiar.

> From here, the 🖥️ for every command in Part B is: **type it on the CP machine, then type the same thing on the W1 machine.**

### B1 — Turn off swap

```bash
sudo swapoff -a
```

📖 **STORY:** **Swap** is when Linux pretends your disk is extra RAM. Kubernetes' scheduler makes promises based on *real* memory ("this truck has 4 GB"). If the OS secretly swaps to disk, those promises become lies and performance gets unpredictable — so the kubelet flatly refuses to start while swap is on. This command disables it right now.

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

📖 **STORY:** The previous command only disables swap until the next reboot. `/etc/fstab` is the file that re-enables it on every boot. `sed` is a stream editor; this line finds the swap entry and comments it out (`#`) so swap stays off permanently. (On most Azure Ubuntu images there's no swap line anyway — this just guarantees it.)

### B2 — Load two kernel modules for container networking

```bash
sudo modprobe overlay
```

📖 **STORY:** `modprobe` loads a kernel module (a driver). **overlay** is the filesystem technology that lets containers stack their layers efficiently — it's how a container image's read-only layers and a writable top layer become one filesystem. The runtime needs it.

```bash
sudo modprobe br_netfilter
```

📖 **STORY:** **br_netfilter** lets the Linux firewall (iptables) *see* traffic that crosses a network bridge. Pods talk to each other over a virtual bridge, so without this, the firewall rules that route pod traffic would be invisible and networking would silently break.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

📖 **STORY:** Same permanence problem as swap: a reboot would unload those modules. This writes their names into a config file that Linux reads at every boot, so they load automatically forever. `cat <<EOF ... EOF` is a **here-document** — a way to feed several lines into a command; `tee` writes those lines to the file (and `sudo` lets it write to a protected location).

### B3 — Turn on the kernel networking switches

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

📖 **STORY:** These are **sysctl** settings — knobs that change kernel behavior. `bridge-nf-call-iptables=1` says "yes, apply firewall rules to bridged pod traffic" (the partner to the module above). `ip_forward=1` turns your machine into a router so it can pass packets *between* pods and nodes — without it, a pod on one node could never reach a pod on another. We write them to a file so they persist.

```bash
sudo sysctl --system
```

📖 **STORY:** Writing the file isn't enough; this command tells the kernel to read all sysctl files now and apply them immediately, so you don't have to reboot.

### B4 — Install and configure the container runtime (containerd)

```bash
sudo apt-get update
```

📖 **STORY:** Refreshes Ubuntu's catalog of available software so the next install pulls current versions.

```bash
sudo apt-get install -y containerd
```

📖 **STORY:** **containerd** is the engine that actually pulls images and runs containers. Kubernetes itself doesn't run containers — it delegates to a runtime through a standard interface (the CRI). `containerd` is the industry-standard choice. (`-y` auto-confirms the prompt.)

```bash
sudo mkdir -p /etc/containerd
```

📖 **STORY:** Makes the folder where containerd's config lives. `-p` means "don't complain if it already exists."

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

📖 **STORY:** containerd ships with no config file, just sensible built-in defaults. This prints those defaults and saves them to a real file we can now edit. (You'll see a wall of text scroll by — that's the full default config being written.)

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

📖 **STORY:** This is the one setting that trips everyone up. A **cgroup** is how Linux limits a process's CPU/memory. Both the kubelet and containerd must agree on *who* manages cgroups, and on modern Ubuntu that manager is **systemd**. This `sed` flips containerd's setting to `systemd`. If the two disagree, the kubelet crash-loops with a confusing error — so we fix it up front.

```bash
sudo systemctl restart containerd
```

📖 **STORY:** Restarts containerd so it picks up the config change we just made.

```bash
sudo systemctl enable containerd
```

📖 **STORY:** `enable` means "start automatically on every boot." Now the engine is always running.

✅ **CHECK (do not skip — this is the #1 silent failure):**
```bash
sudo grep SystemdCgroup /etc/containerd/config.toml
```
It must print `SystemdCgroup = true`. If it still says `false`, your `sed` didn't apply (a common cause: forgetting the closing `/` in `s/.../.../`). Re-run the `sed` exactly as written, then `sudo systemctl restart containerd`, then grep again. If the kubelet and containerd disagree on the cgroup driver, `kubeadm init` appears to work but the kubelet crash-loops.

### B5 — Add Kubernetes' software repository

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

📖 **STORY:** Small helpers so apt can download over HTTPS and verify signatures: `curl` fetches files, `gpg` checks cryptographic signatures, `ca-certificates` holds the trusted authorities.

```bash
sudo mkdir -p /etc/apt/keyrings
```

📖 **STORY:** A folder to store the key that proves Kubernetes packages are authentic.

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

📖 **STORY:** Downloads Kubernetes' signing key and converts it (`--dearmor`) into the binary form apt expects. From now on, apt will only trust Kubernetes packages signed by this key — so nobody can slip you a fake. We're pinning to version line **v1.30** (a stable release).

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

📖 **STORY:** Adds the Kubernetes repository to apt's list of sources, telling it to verify everything from there with the key we just saved. Now `apt` knows *where* to get Kubernetes and *how* to trust it.

### B6 — Install the three Kubernetes tools

```bash
sudo apt-get update
```

📖 **STORY:** Refresh again so apt sees the new Kubernetes repository.

```bash
sudo apt-get install -y kubelet kubeadm kubectl
```

📖 **STORY:** The three tools, each with a clear job:
- **kubelet** — the agent on every node; it talks to the control plane and makes sure the containers it's told to run are actually running. It's the "driver" in our trucking story.
- **kubeadm** — the one-time setup tool that bootstraps the cluster (we use it in Part C).
- **kubectl** — *your* remote control for talking to the cluster ("kube-control"). You'll live in this command.

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

📖 **STORY:** "hold" freezes these three at their current version. Kubernetes is picky about version differences between components; a stray `apt upgrade` that bumped only one of them could break the cluster. Holding them means upgrades become a deliberate choice, never an accident.

```bash
sudo systemctl enable --now kubelet
```

📖 **STORY:** Enables the kubelet to start on boot and starts it now. It'll actually keep restarting and waiting at this point — that's normal. It has nothing to do until Part C tells it about the cluster.

✅ **CHECK (run on each machine):**
```bash
kubeadm version
```
```bash
systemctl is-active containerd
```
The first prints something like `v1.30.x`; the second prints `active`. When both machines pass, **Part B is done** and your kitchens are prepped. Now go back and make sure you did all of B1–B6 on **both** the CP and W1 machines before continuing.

---

# PART C — Bring the cluster to life

📖 **STORY:** Both kitchens are prepped. Now we open the dispatch office for business (`kubeadm init` on the control-plane), give the office a way to reach every truck's parcels (the pod network), and put one truck on the road (`kubeadm join` on the worker). After this part, you have a living cluster.

### C1 — Initialize the control plane

🖥️ **WHERE:** the **CP** machine only.

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<CP_PRIVATE> \
  --pod-network-cidr=192.168.0.0/16
```

📖 **STORY:** This is the big moment. `kubeadm init` builds the dispatch office: it generates all the TLS certificates the components use to trust each other, pulls the control-plane container images, and starts the four clerks (API server, etcd, scheduler, controller-manager) as special "static pods."
- `--apiserver-advertise-address=<CP_PRIVATE>` — tells the API server to listen on the *private* VNet address, so the worker can reach it over the private street. Type the control-plane's **private** IP here, not the public one.
- `--pod-network-cidr=192.168.0.0/16` — reserves a block of IP addresses that will be handed out to Pods. We pick this exact range because the pod-network plugin we install next (Calico) expects it by default. They must match.

When it finishes, it prints a **`kubeadm join ...` command**. 📌 **Copy that entire line into your notes now** — the worker needs it in C3, and it contains a one-time token.

### C2 — Let your user run `kubectl`

🖥️ **WHERE:** the **CP** machine.

```bash
mkdir -p $HOME/.kube
```

📖 **STORY:** `kubectl` looks for a config file describing *which* cluster to talk to and *how* to authenticate. This makes the folder for it in your home directory.

```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

📖 **STORY:** `kubeadm` wrote an admin credentials file at `/etc/kubernetes/admin.conf` (owned by root). We copy it to where `kubectl` looks. This file is essentially the master key to your cluster — treat it like a password.

```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

📖 **STORY:** The copy is still owned by root. `chown` changes its owner to *you* (`$(id -u)` is your user ID, `$(id -g)` your group ID) so `kubectl` can read it without `sudo`.

✅ **CHECK:**
```bash
kubectl get nodes
```
You'll see `k8s-cp` with status **NotReady**. That's *correct right now* — the office is open, but there's no pod network yet, so it's not ready to place work. We fix that next.

```bash
kubectl get pods -A
```
📖 **STORY:** `-A` means "all namespaces." You'll see the control-plane pods Running, but **coredns** stuck Pending — it's politely waiting for a network, which is our next step.

### C3 — Install the pod network (Calico)

🖥️ **WHERE:** the **CP** machine.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

📖 **STORY:** Pods need IP addresses and a way to reach each other across nodes. Kubernetes leaves that job to a pluggable component called a **CNI** (Container Network Interface). **Calico** is the most common one. `kubectl apply -f <url>` downloads Calico's manifest and tells the cluster to create everything in it. Think of this as laying the internal road network between all parcels, on every truck.

⚠️ **AZURE-CRITICAL — switch Calico to VXLAN immediately after applying.** Calico's default manifest uses **IP-in-IP** encapsulation, which **Azure silently drops between VMs**. Without this, single-node works but all *cross-node* pod traffic (including DNS to CoreDNS) fails with "Could not resolve host". Run this right after the apply:
```bash
kubectl patch ippool default-ipv4-ippool --type=merge -p '{"spec":{"ipipMode":"Never","vxlanMode":"Always"}}'
kubectl -n kube-system rollout restart daemonset/calico-node
```

```bash
kubectl get pods -n kube-system -w
```

📖 **STORY:** `-w` means "watch" — it keeps printing updates live. Wait until `calico-node` and `coredns` show **Running** (a minute or two), then press **Ctrl-C** to stop watching.

✅ **CHECK:**
```bash
kubectl get nodes
```
`k8s-cp` now says **Ready**. The road network is laid; the office is fully operational.

### C4 — Put the worker on the road

🖥️ **WHERE:** the **W1** machine.

```bash
sudo kubeadm join <CP_PRIVATE>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

📖 **STORY:** Paste the *exact* line `kubeadm init` gave you in C1 (yours will have a real token and hash). It tells the worker: "here's the office address (`CP_PRIVATE:6443` — 6443 is the API server's port), here's a one-time token proving I'm allowed to join, and here's the fingerprint so I know I'm talking to the *real* office and not an impostor." The worker's kubelet then registers itself with the control plane.

> 🆘 Lost the join command, or token expired (they last 24h)? On the **CP** machine run:
> ```bash
> kubeadm token create --print-join-command
> ```
> It prints a fresh, ready-to-paste join line.

✅ **CHECK (on the CP machine):**
```bash
kubectl get nodes -o wide
```
Both `k8s-cp` and `k8s-w1` show **Ready**. 🎉 **You just built a real Kubernetes cluster by hand.** Sit with that for a second — most candidates have never done this.

```bash
kubectl label node k8s-w1 node-role.kubernetes.io/worker=worker
```
📖 **STORY:** Optional cosmetics: tags the worker with a role label so `kubectl get nodes` shows "worker" in the ROLES column. **Labels** are just key=value tags on objects — and they turn out to be how Kubernetes wires things together, as you'll see with Services.

---

# PART D — Run your app (and learn the core objects by typing them)

📖 **STORY:** Now we deploy an app — but more importantly, you'll hand-write the five objects that make up 90% of all Kubernetes work: a **Namespace**, a **ConfigMap**, a **Secret**, a **Deployment**, and a **Service**. We describe *desired state* in YAML files and hand them to the cluster; the controllers make it real and keep it real.

🖥️ **WHERE:** everything in Part D and beyond is on the **CP** machine (it has `kubectl`).

> **Type each YAML file by hand** with `nano`, exactly as shown. YAML is whitespace-sensitive: use **spaces, never tabs**, and keep the indentation identical. After writing each file, you apply it with `kubectl apply -f <file>`.

### D1 — A Namespace (a room of your own)

```bash
nano 00-namespace.yaml
```
Type this in:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```
📖 **STORY line by line:**
- `apiVersion: v1` — which version of the Kubernetes API this object speaks. Core objects like Namespace/Pod/Service are `v1`.
- `kind: Namespace` — *what* you're creating.
- `metadata.name: demo` — its name. A **Namespace** is a virtual room that isolates your stuff from everything else in the cluster, so your `web` doesn't collide with someone else's `web`. Real teams use namespaces to separate dev/staging/prod.

Apply it:
```bash
kubectl apply -f 00-namespace.yaml
```
📖 **STORY:** `apply` means "make the cluster match this file." If the object doesn't exist, it's created; if it exists, it's updated to match. This declarative style — *describe the end state, let Kubernetes converge to it* — is the whole philosophy.

### D2 — A ConfigMap and a Secret (configuration, separated from code)

```bash
nano 10-config.yaml
```
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
data:
  APP_GREETING: "Hello from Nifty's kubeadm cluster"
  APP_ENV: "lab"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
type: Opaque
stringData:
  API_TOKEN: "super-secret-token-change-me"
```
📖 **STORY:**
- The `---` is a YAML separator letting us put **two objects in one file**.
- A **ConfigMap** holds non-secret configuration as key/value pairs. Why? So you never bake settings into your image — the same image runs in dev and prod, only the ConfigMap differs.
- A **Secret** is the same idea for *sensitive* values (passwords, tokens, keys). It looks almost identical, but Kubernetes treats it specially: it can be locked down with access rules and encrypted at rest. `type: Opaque` just means "arbitrary user data." `stringData` lets you type the value in plain text and Kubernetes base64-encodes it for you.
- Note both live in `namespace: demo` — the room we made.

```bash
kubectl apply -f 10-config.yaml
```

### D3 — A Deployment (the promise: "always keep 3 of these running")

```bash
nano 20-deployment.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginxdemos/hello:plain-text
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: app-config
          env:
            - name: API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: API_TOKEN
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
```
📖 **STORY — this is the most important object, so let's walk it:**
- `kind: Deployment`, `apiVersion: apps/v1` — Deployments live in the `apps` API group.
- `spec.replicas: 3` — your **desired state**: always 3 copies. This is the promise the controller-manager enforces forever.
- `selector.matchLabels: app=web` — how the Deployment knows *which* Pods are "its" Pods: the ones labelled `app: web`. **Labels are the glue.**
- `template` — the cookie-cutter for each Pod. Everything under here describes one Pod, stamped out 3 times.
  - `image: nginxdemos/hello:plain-text` — a tiny demo web app that prints *which pod* answered. Perfect for *seeing* load balancing and self-healing.
  - `envFrom.configMapRef` — inject every key from our ConfigMap as environment variables. This is config-from-outside in action.
  - `env ... secretKeyRef` — inject one value from the Secret as `API_TOKEN`. Same idea, for sensitive data.
  - `readinessProbe` — Kubernetes keeps checking `GET /` ; until it succeeds, this Pod receives **no traffic**. Prevents sending users to a Pod that's still starting.
  - `livenessProbe` — if `GET /` later starts failing (the app hung), Kubernetes **restarts** the container. Self-healing at the process level.
  - `resources.requests` — what the scheduler *reserves* for the Pod (used to decide which node has room). `resources.limits` — the hard ceiling; exceed the memory limit and the Pod is killed and restarted. This is how one bad app can't starve the others.

```bash
kubectl apply -f 20-deployment.yaml
```
```bash
kubectl -n demo get pods -o wide
```
📖 **STORY:** `-n demo` scopes the command to our room; `-o wide` adds columns like which node each Pod landed on. Watch 3 `web-...` Pods appear and go `Running`, possibly spread across both nodes — the scheduler decided placement.

✅ **Prove the config injection worked:**
```bash
POD=$(kubectl -n demo get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
```
📖 **STORY:** This grabs the name of the first `web` Pod into a variable. `-l app=web` filters by label; `-o jsonpath=...` plucks one field out of the JSON.
```bash
kubectl -n demo exec $POD -- printenv | grep -E "APP_GREETING|APP_ENV|API_TOKEN"
```
📖 **STORY:** `exec ... -- printenv` runs `printenv` *inside* the Pod and shows its environment variables. You'll see your ConfigMap and Secret values living inside the container — exactly as injected.

### D4 — A Service (one stable address in front of changing Pods)

```bash
nano 30-service.yaml
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: demo
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```
📖 **STORY:** Pods are mortal — they get deleted, recreated, and change IPs constantly. So how does anything find them reliably? A **Service** is a stable, never-changing virtual IP and DNS name that *automatically* tracks all healthy Pods matching its `selector` (here, `app: web`) and load-balances across them. `type: ClusterIP` means it's reachable only *inside* the cluster — which is what we want, because the Ingress (Part E) will be the public door. `port` is the Service's port; `targetPort` is the Pod's port it forwards to.

```bash
kubectl apply -f 30-service.yaml
```

✅ **See load balancing with your own eyes:**
```bash
kubectl -n demo run curl --rm -it --image=curlimages/curl --restart=Never -- sh -c 'for i in 1 2 3 4 5 6; do curl -s web.demo.svc.cluster.local | grep -i "server name"; done'
```
📖 **STORY:** This spins up a throwaway Pod with `curl` inside the cluster and hits the Service's DNS name (`web.demo.svc.cluster.local` — the standard `service.namespace.svc.cluster.local` form) six times. Watch the **server name change** between Pods — that's the Service spreading requests across your 3 replicas. `--rm` deletes the throwaway Pod when done.

📖 **THE LADDER (say this in the interview):** *Pod* (one running unit) ← kept in triplicate by → *ReplicaSet* ← managed for rolling updates by → *Deployment*. And *Service* sits in front, giving them one stable address by matching **labels**. Next, *Ingress* sits in front of the Service to bring in public HTTPS traffic.

---

# PART E — Open the public door with Ingress + real HTTPS

📖 **STORY:** Your app works *inside* the cluster. Now we let the world in — securely. Two new players:
1. An **Ingress controller** (we'll use **ingress-nginx**): a reverse proxy running *inside* the cluster that listens on ports 80/443 and routes incoming web requests to the right Service based on hostname. Because we have no cloud load balancer (this is self-managed) and only 80/443 are open, we make the controller bind directly to the worker's ports using `hostNetwork`.
2. **cert-manager**: a robot that talks to **Let's Encrypt**, proves you own `chishty.me`, and installs a real TLS certificate automatically — then renews it forever.

### E1 — Install Helm (the package manager for Kubernetes)

🖥️ **WHERE:** the **CP** machine.
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
📖 **STORY:** Some software (like ingress-nginx) is dozens of objects. **Helm** is "apt for Kubernetes" — it installs a whole bundle (a "chart") with one command and sensible settings. This downloads and installs the `helm` tool.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```
📖 **STORY:** Registers the official ingress-nginx chart repository with Helm, under the nickname `ingress-nginx`.

```bash
helm repo update
```
📖 **STORY:** Refreshes Helm's catalog so it knows the latest chart versions.

### E2 — Install the ingress controller, bound to the node's ports

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.kind=DaemonSet \
  --set controller.hostNetwork=true \
  --set controller.dnsPolicy=ClusterFirstWithHostNet \
  --set controller.service.type=ClusterIP
```
📖 **STORY — the flags are the whole lesson here:**
- `helm install <name> <chart>` — install the chart, calling this release `ingress-nginx`.
- `-n ingress-nginx --create-namespace` — put it in its own room, creating that room.
- `controller.kind=DaemonSet` — run one copy on *every* node (a **DaemonSet** = "one Pod per node"), so any node can receive web traffic.
- `controller.hostNetwork=true` — the key trick: let the controller use the **node's own network**, binding directly to the host's `:80` and `:443`. Normally pods are isolated from the host network; this opts in, which is how requests from the internet (allowed only on 80/443) reach it without a cloud load balancer.
- `controller.dnsPolicy=ClusterFirstWithHostNet` — because we're on the host network, this keeps the controller able to use the cluster's internal DNS.
- `controller.service.type=ClusterIP` — we don't ask Azure for a load balancer (we'd never get one on a self-managed cluster); traffic comes straight to the host ports instead.

✅ **CHECK:**
```bash
kubectl -n ingress-nginx get pods -o wide
```
The controller Pod is `Running` on `k8s-w1`. Now test from your **laptop**:
```bash
curl -I http://<W1_PUBLIC>
```
📖 **STORY:** You should get `HTTP/1.1 404 Not Found` from nginx. **A 404 here is success!** It means nginx is alive and listening on the worker's port 80 — it just has no route defined yet. (`-I` asks for headers only.)

### E3 — Point your domain at the worker

🖥️ **WHERE:** your **DNS provider's** website (wherever chishty.me is managed).

Create an **A record**:
```
Type: A     Name: k8s     Value: <W1_PUBLIC>     TTL: 300
```
📖 **STORY:** An **A record** maps a name to an IPv4 address. This makes `k8s.chishty.me` resolve to your worker's public IP, so when anyone (including Let's Encrypt) visits that name, they arrive at your ingress controller. Low TTL (300s) means changes propagate fast.

✅ **CHECK (from your laptop, after a minute or two):**
```bash
dig +short k8s.chishty.me
```
It should print `<W1_PUBLIC>`. If it's blank, DNS hasn't propagated yet — wait and retry. **Do this step early**, since propagation can take a few minutes.

### E4 — Install cert-manager

🖥️ **WHERE:** the **CP** machine.
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.0/cert-manager.yaml
```
📖 **STORY:** Installs the cert-manager robot and the new object types it understands (like `ClusterIssuer` and `Certificate`). It will watch for Ingresses that want certificates and handle the whole Let's Encrypt dance.

```bash
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
```
📖 **STORY:** `rollout status` blocks until that component is fully Ready. We wait for the webhook specifically because the next step talks to it; applying too early gives a confusing error.

### E5 — Tell cert-manager how to get certificates (the ClusterIssuer)

```bash
nano 50-clusterissuer.yaml
```
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: apps.techenrage@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```
📖 **STORY:**
- A **ClusterIssuer** is a cluster-wide recipe for *where* and *how* to obtain certificates.
- `acme.server` — the Let's Encrypt production API (ACME is the protocol for automated certs).
- `email` — Let's Encrypt emails you about expiries; use a real address.
- `solvers: http01` — *how* you'll prove you own the domain: the **HTTP-01 challenge**. Let's Encrypt will request a secret file at `http://k8s.chishty.me/.well-known/acme-challenge/...`; cert-manager temporarily serves it through the nginx ingress. This is exactly why port 80 had to be open.

```bash
kubectl apply -f 50-clusterissuer.yaml
```

### E6 — Create the Ingress (this triggers the certificate)

```bash
nano 40-ingress.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: demo
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - k8s.chishty.me
      secretName: web-tls
  rules:
    - host: k8s.chishty.me
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```
📖 **STORY:**
- An **Ingress** is the routing rule: "requests for host `k8s.chishty.me`, path `/`, go to the `web` Service on port 80."
- `ingressClassName: nginx` — handle this with the ingress-nginx controller we installed.
- The `cert-manager.io/cluster-issuer` annotation is the magic word: cert-manager sees it, notices the `tls` block wants a cert for `k8s.chishty.me` stored in a Secret called `web-tls`, and goes off to get it from Let's Encrypt automatically.
- `tls.secretName: web-tls` — where the issued certificate will be stored (cert-manager creates this Secret for you).

```bash
kubectl apply -f 40-ingress.yaml
```

✅ **Watch the certificate get issued:**
```bash
kubectl -n demo get certificate
```
📖 **STORY:** `READY` starts as `False`, then flips to `True` within a minute or two once the challenge succeeds. Watching it work:
```bash
kubectl -n demo describe certificate web-tls
```
The Events at the bottom narrate the whole issuance (or tell you what's stuck).

✅ **THE PAYOFF — from your laptop:**
```bash
curl -I https://k8s.chishty.me
```
`HTTP/2 200` with a valid certificate. Now open **https://k8s.chishty.me** in a browser: padlock 🔒 and your app, served from one of the Pods. **Screenshot this** — it's your portfolio centerpiece and your interview demo.

📖 **The full request path to recite:** browser → DNS resolves `k8s.chishty.me` to the worker IP → arrives at the worker's `:443` (ingress-nginx via hostNetwork) → TLS terminated using the `web-tls` cert → Ingress rule matches the host → forwards to the `web` Service → Service load-balances to one healthy Pod. You built every hop.

---

# PART F — Operate it like the job (day-2 commands)

📖 **STORY:** Building a cluster is day one. The actual job is *running* it: healing failures, shipping updates safely, scaling, and debugging. Each command below is a "have you ever..." interview question you'll now answer with "yes, here's how."

### Self-healing — kill a Pod, watch it come back
```bash
kubectl -n demo get pods
```
```bash
kubectl -n demo delete pod <one-web-pod-name>
```
```bash
kubectl -n demo get pods -w
```
📖 **STORY:** You deleted a Pod; the Deployment promised 3; the controller-manager instantly notices "only 2 — that violates desired state" and creates a replacement. Press Ctrl-C when you've seen it. *This is reconciliation, live.*

### Rolling update — ship a new version with zero downtime
```bash
kubectl -n demo set image deploy/web web=nginxdemos/hello:0.3-plain-text
```
```bash
kubectl -n demo rollout status deploy/web
```
📖 **STORY:** `set image` changes the container image. The Deployment then does a **rolling update**: it brings up new Pods, waits for their readiness probes, and only then removes old ones — so users never see an outage. `rollout status` shows the progress.

```bash
kubectl -n demo rollout history deploy/web
```
📖 **STORY:** Kubernetes keeps a revision history of the Deployment — your deploy log, built in.

### Rollback — undo a bad deploy instantly
```bash
kubectl -n demo rollout undo deploy/web
```
📖 **STORY:** Reverts to the previous revision. In a real incident, this one command is how you stop the bleeding while you investigate. Tell interviewers: "rollback is `rollout undo`, or redeploy the previous image tag."

### Scaling — by hand, then automatically
```bash
kubectl -n demo scale deploy/web --replicas=5
```
```bash
kubectl -n demo get pods
```
📖 **STORY:** Manually changed desired state to 5; two more Pods appear. Now make it automatic.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
📖 **STORY:** The **metrics-server** collects CPU/memory usage from each node — the data autoscaling and `kubectl top` need.

```bash
kubectl -n kube-system patch deploy metrics-server --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```
📖 **STORY:** On a self-built cluster the kubelets use self-signed certificates, which metrics-server distrusts by default and fails. This `patch` adds a flag telling it to accept them in our lab. (`patch` edits a live object in place.)

```bash
kubectl top nodes
```
```bash
kubectl top pods -n demo
```
📖 **STORY:** If these print CPU/memory numbers, metrics work. Now add the autoscaler:

```bash
nano 60-hpa.yaml
```
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
  namespace: demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```
📖 **STORY:** A **HorizontalPodAutoscaler (HPA)** watches the average CPU of the `web` Pods and adjusts the replica count between 2 and 6 to keep CPU near 50%. Traffic up → more Pods; quiet → fewer. This is elasticity, declared in 18 lines.
```bash
kubectl apply -f 60-hpa.yaml
```
```bash
kubectl -n demo get hpa
```

### The debugging reflexes — drill these until automatic
```bash
kubectl -n demo get pods
```
📖 **STORY:** Always start with status. The STATUS column names the problem.
```bash
kubectl -n demo describe pod <pod>
```
📖 **STORY:** Scroll to **Events** at the bottom — they explain ~90% of failures in plain English ("failed to pull image", "insufficient cpu", "probe failed").
```bash
kubectl -n demo logs <pod>
```
📖 **STORY:** The app's own output. Add `-p` (`logs -p <pod>`) to see logs from a *previous*, crashed container — essential for `CrashLoopBackOff`.
```bash
kubectl -n demo exec -it <pod> -- sh
```
📖 **STORY:** Drops you into a shell *inside* the container to poke around (check files, env, connectivity). Type `exit` to leave.

**The four statuses to recognize instantly:**

| You see | It means | First move |
|---|---|---|
| `ImagePullBackOff` | Can't fetch the image (typo'd name/tag, or private registry needs a pull secret) | `describe pod`, check the image string |
| `CrashLoopBackOff` | Container starts then exits repeatedly (bad config/command) | `logs -p` |
| `Pending` | No node has room for the Pod's `requests`, or a taint blocks it | `describe pod` events |
| `Running` but `0/1 READY` | readiness probe failing | `describe pod`, check the probe |

✅ **Checkpoint F:** you've demonstrated self-healing, rolling update, rollback, manual + auto scaling, and the debug loop. That *is* the day-to-day of the job.

---

# Tear down when you're done (stop billing)
🖥️ **WHERE:** your laptop.
```bash
az group delete -n rg-k8s-lab --yes --no-wait
```
📖 **STORY:** Deletes the whole folder — both VMs, the network, the gate — in one shot. Then remove the `k8s` A record at your DNS provider. Your cluster is gone and the meter is off. (You can rebuild it anytime by retyping this walkthrough — and doing it twice is how it becomes second nature.)

---

# Keep your record (do this as you go)
After each part, jot what happened into your repo so the learning compounds and you have proof:
- Paste `kubectl get nodes` output and your browser screenshot into this lab's `README.md` checklist.
- Add a dated line to `02-troubleshooting.md`.
- **Every error you hit → one entry in `02-troubleshooting.md`** (symptom → cause → fix). That file is the single most convincing thing you can show an interviewer.

**Whenever you get stuck, paste me the exact command and the exact error. I'll explain what went wrong in plain language, get you unstuck, and hand you a ready-made entry for `ISSUES-AND-FIXES.md`.** That's how we'll work through tonight together.

