# Troubleshooting Log — real failures, root-caused & fixed

> Every error I actually hit while building this cluster, and how I diagnosed and fixed it. This is the most useful part of the repo: it shows I debug **layer by layer** instead of just following a tutorial. Each entry follows: *Context → Symptom → Root cause → Fix → Lesson.*

**Quick index**
1. SSH to new VMs times out — NSG rule-name collision
2. containerd cgroup driver silently wrong — broken `sed`
3. `kubeadm join` fails — needs `sudo`
4. Pod DNS fails cross-node — Calico IP-in-IP blocked on Azure (→ VXLAN)
5. `ImagePullBackOff` — non-existent image tag + how `rollout undo` behaves

---
### 2026-06-14 — SSH to new Azure VMs times out (port 22)
**Context:** Finished Part A of Lab 04; tried `ssh controlplane@<public-ip>` to both kubeadm VMs.
**Symptom:** `ssh: connect to host <ip> port 22: Operation timed out` on both VMs. (Timeout, not "connection refused" → traffic is being dropped by a firewall, not by the host.)
**Root cause:** I created all three NSG rules with the **same name** `-n allow-ssh`. In Azure the rule name is its unique key, so each `az network nsg rule create` **overwrote** the previous rule instead of adding one. Net result: a single rule that only opened 443. Ports 22 and 80 were never open.
**Fix:**
```bash
az network nsg rule delete -g $RG --nsg-name $NSG -n allow-ssh
az network nsg rule create -g $RG --nsg-name $NSG -n allow-ssh   --priority 100 --destination-port-ranges 22  --access Allow --protocol Tcp --direction Inbound
az network nsg rule create -g $RG --nsg-name $NSG -n allow-http  --priority 110 --destination-port-ranges 80  --access Allow --protocol Tcp --direction Inbound
az network nsg rule create -g $RG --nsg-name $NSG -n allow-https --priority 120 --destination-port-ranges 443 --access Allow --protocol Tcp --direction Inbound
az network nsg rule list -g $RG --nsg-name $NSG -o table   # verify 3 distinct rules
```
**Lesson:** Every NSG rule needs a **unique name** — reusing a name silently replaces the old rule. Also: an SSH **timeout** points to a network/firewall block (NSG), whereas **connection refused** points to the host (sshd down / wrong port). Diagnose by which one you get.
**Bonus lesson:** These VMs landed in a **shared** resource group (`rg-ai-experimental-env`) with other people's servers — so cleanup must target only my VMs, never `az group delete`.

---
### 2026-06-14 — containerd cgroup driver stayed `false` (broken `sed`)
**Context:** Node prep (Part B) on the control plane — setting the containerd cgroup driver to systemd.
**Symptom:** `sed: -e expression #1, char 44: unterminated 's' command`. The command errored and made no change, but I almost moved on without noticing.
**Root cause:** I typed `'s/SystemdCgroup = false/SystemdCgroup = true'` — missing the closing `/`. A `sed` substitution must be `s/find/replace/`. Without the final delimiter sed rejects the whole expression, so containerd kept `SystemdCgroup = false`.
**Why it matters:** kubelet uses the `systemd` cgroup driver on modern Ubuntu. If containerd uses `cgroupfs` instead, the two disagree and the kubelet crash-loops shortly after `kubeadm init` — with an error that doesn't obviously point at cgroups.
**Fix:**
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml   # note trailing /
sudo systemctl restart containerd
sudo grep SystemdCgroup /etc/containerd/config.toml   # must show: SystemdCgroup = true
```
**Lesson:** Always *verify* a config edit landed (`grep` it) instead of trusting that the command ran. containerd installed as v2.2.1 here (config `version = 3`), and the `SystemdCgroup` key is still the correct one to flip.

---
### 2026-06-14 — `kubeadm join` failed: "user is not running as root"
**Context:** Joining the worker node to the cluster with the `kubeadm join` line from `kubeadm init`.
**Symptom:** `[ERROR IsPrivilegedUser]: user is not running as root`.
**Root cause:** I copied the join command without `sudo`. `kubeadm join` writes to `/var/lib/kubelet` and `/etc/kubernetes` and starts the kubelet — all root-only.
**Fix:** Prefix with `sudo`: `sudo kubeadm join 10.20.1.4:6443 --token <t> --discovery-token-ca-cert-hash sha256:<h>`.
**Lesson:** The join command printed by `kubeadm init` says "run the following on each as root" — it assumes a root shell. From a normal user, add `sudo`. (Also: the token expires after 24h; regenerate with `kubeadm token create --print-join-command`.)

---
### 2026-06-14 — Deployment rejected: `envForm`/`valueForm` + a CPU-units trap
**Context:** Applying the `web` Deployment manifest (Part D).
**Symptom:** `strict decoding error: unknown field "spec.template.spec.containers[0].env[0].valueForm", unknown field "...envForm"`.
**Root cause:** Typos — the correct keys are `envFrom` (import all keys of a ConfigMap as env vars) and `valueFrom` (pull one value from a Secret/ConfigMap). Kubernetes does *strict* decoding and rejects unknown fields, which is what surfaced the typo.
**Hidden second bug:** I also wrote `cpu: "50"` instead of `cpu: "50m"`. `50m` = 50 millicores = 0.05 CPU; `"50"` = 50 whole CPUs, which no 2-core node can satisfy — that would leave every pod stuck in `Pending` with no error from `kubectl apply`.
**Fix:**
```bash
sed -i 's/envForm/envFrom/; s/valueForm/valueFrom/; s/cpu: "50"/cpu: "50m"/' 20-deployment.yaml
kubectl apply -f 20-deployment.yaml
```
**Lesson:** Two reflexes — (1) trust strict-decoding errors, they name the exact bad field; (2) CPU is in millicores (`m`); a missing `m` is a 1000× mistake and a classic cause of `Pending` pods.

---
### 2026-06-14 — Pod DNS fails cross-node ("Could not resolve host") — Calico IP-in-IP blocked on Azure
**Context:** App pods Running 1/1 and the Service had 3 endpoints, but a test pod couldn't reach the Service by DNS.
**Symptom:** `curl -v http://web.demo.svc.cluster.local` → `Could not resolve host` (curl exit 6). `kubectl get endpoints web` showed all 3 backends, so the Service was fine.
**Root cause:** Cross-node pod networking was broken. CoreDNS runs on the control-plane node; the app/test pods run on the worker. So DNS queries must cross nodes — and Calico's default manifest uses **IP-in-IP encapsulation, which Azure's network silently drops between VMs**. Single-node traffic worked; cross-node traffic (incl. reaching CoreDNS) was black-holed.
**Fix:** Switch Calico to VXLAN encapsulation (the cloud-friendly mode):
```bash
kubectl patch ippool default-ipv4-ippool --type=merge -p '{"spec":{"ipipMode":"Never","vxlanMode":"Always"}}'
kubectl -n kube-system rollout restart daemonset/calico-node
kubectl -n kube-system rollout restart deployment/coredns
kubectl -n demo rollout restart deployment/web
```
**Lesson:** On Azure (and several clouds) **Calico must use VXLAN, not IP-in-IP** — the cloud fabric doesn't forward IPIP between VMs. Diagnosis flow that nailed it: Service had endpoints (Service OK) → DNS name wouldn't resolve (CoreDNS unreachable) → CoreDNS was on a different node (cross-node networking) → encapsulation mismatch. Great interview story about debugging layer by layer.

---
### 2026-06-16 — `ImagePullBackOff` from a non-existent image tag (and `rollout undo` rolled back INTO it)
**Context:** Practicing rolling updates on the `web` Deployment.
**Symptom:** New pods stuck `ErrImagePull` → `ImagePullBackOff`; `describe pod` Events: `failed to resolve image: docker.io/nginxdemos/hello:3.0-plain-text: not found`.
**Root cause:** I set the image to tag `3.0-plain-text`, which doesn't exist (valid tags are `plain-text`, `0.3-plain-text`). The deployment couldn't pull it, so the rollout stalled with old pods still up (good — that's rolling-update safety). Then `kubectl rollout undo` rolled back to the *previous* revision, which happened to also be the broken `3.0` one, so it stayed stuck.
**Fix:** Set a valid tag and let it converge:
```bash
kubectl -n demo set image deploy/web web=nginxdemos/hello:0.3-plain-text
kubectl -n demo rollout status deploy/web
```
**Lesson:** `ImagePullBackOff` almost always = wrong image name/tag or a private registry without pull creds — always read `describe pod` Events for the exact reason. Also: a rolling update with a bad image **does not** take down the running version (readiness gating protects you), and `rollout undo` goes to the *previous revision* — check `rollout history` first so you don't roll back into another bad one.
