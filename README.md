# 🔹 Step 1 — Prepare Host OS (Master & Worker Nodes)

## 1. Update Sistem & Tools Dasar

Selalu pastikan semua paket OS sudah up-to-date.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget gnupg lsb-release apt-transport-https ca-certificates
```

## 2. Set Hostname (unik untuk tiap node)

Gunakan `hostnamectl` agar nama host konsisten.

```bash
# Master Node
sudo hostnamectl set-hostname master-node

# Worker Node 1
sudo hostnamectl set-hostname worker-node1

# Worker Node 2
sudo hostnamectl set-hostname worker-node2
```

👉 **Kenapa penting?**

* Kubernetes heavily relies on hostname & resolusi DNS/hosts.
* Kalau salah, node bisa stuck di status `NotReady`.

---

## 3. Konfigurasi Static IP & /etc/hosts

Pastikan tiap VM pakai IP statis (via DHCP reservation atau manual).
Tambahkan mapping di `/etc/hosts` **pada semua node**:

```bash
sudo nano /etc/hosts
```

Isi (contoh):

```
192.168.10.100 master-node
192.168.10.101 worker-node1
192.168.10.102 worker-node2
```

👉 **Kenapa?**

* Supaya `kubelet` dan `kubeadm` bisa resolve antar node tanpa DNS server eksternal.

---

## 4. Disable Swap

Kubernetes **tidak mendukung swap**.

```bash
# Matikan swap segera
sudo swapoff -a

# Nonaktifkan swap permanen
sudo sed -i '/ swap / s/^\(.*\)$/#\1/' /etc/fstab
```

Cek lagi:

```bash
free -h
```

Swap harus `0B`.

---

## 5. Sinkronisasi Waktu (Optional tapi Disarankan)

Perbedaan waktu bisa bikin masalah di TLS cert.

```bash
sudo apt install -y chrony
sudo systemctl enable chrony --now
timedatectl
```

---

✅ Setelah Step 1 ini:

* OS sudah siap (update + tools dasar).
* Hostname unik.
* IP statis & resolvable via `/etc/hosts`.
* Swap mati permanen.
* Waktu sinkron.

---

# 🔹 Step 2 — Configure Kernel Modules & Sysctl

## 1. Load Kernel Modules

Kubernetes membutuhkan `overlay` dan `br_netfilter` supaya iptables bisa memproses traffic antar pod.
Jalankan di **semua node (master & worker)**:

```bash
# Load langsung (efek sementara, sampai reboot)
sudo modprobe overlay
sudo modprobe br_netfilter
```

---

## 2. Persist Kernel Modules

Supaya tetap aktif setelah reboot, buat config:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

---

## 3. Sysctl untuk Networking

Atur parameter jaringan supaya pod bisa saling komunikasi:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Apply perubahan:

```bash
sudo sysctl --system
```

---

## 4. Verifikasi

Pastikan nilai sudah sesuai:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

Hasil yang benar:

```
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

---

## 🔎 Catatan untuk Troubleshooting

* Kalau nilai **tidak berubah** setelah reboot → biasanya karena ada conflict di file `/etc/sysctl.conf`. Bisa tambahkan setting langsung di sana sebagai backup.
* Pastikan package `bridge-utils` terinstall kalau mau debug:

  ```bash
  sudo apt install -y bridge-utils
  ```

---

✅ Setelah Step 2 ini:

* Node sudah bisa memproses traffic antar pod.
* Persiapan networking untuk CNI (Flannel/Calico) sudah aman.

---
# 🔹 Step 3 — Install `containerd`

## 1. Kenapa `containerd`, bukan Docker?

* **containerd** adalah runtime murni yang fokus hanya pada menjalankan container (lebih ringan & cepat).
* **Docker** sebenarnya di atas `containerd` → jadi ada overhead ekstra.
* Kubernetes sejak v1.24 resmi **menghapus dukungan langsung untuk Docker** → jadi best practice sekarang pakai `containerd`.

---

## 2. Install `containerd`

Jalankan di **semua node (master & worker)**:

```bash
sudo apt update
sudo apt install -y containerd
```

---

## 3. Buat Config Default

Kita perlu generate config supaya bisa custom:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

---

## 4. Ubah Cgroup Driver → systemd

Secara default, `containerd` pakai **cgroupfs**, tapi Kubernetes lebih stabil dengan **systemd**.
Edit file:

```bash
sudo nano /etc/containerd/config.toml
```

Cari bagian:

```
SystemdCgroup = false
```

Ubah jadi:

```
SystemdCgroup = true
```

🔎 Shortcut pakai sed:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

---

## 5. Restart & Enable

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 6. Verifikasi

Cek status:

```bash
systemctl status containerd
```

Output harus `active (running)` ✅

Tes runtime:

```bash
sudo ctr images ls
```

(jika kosong → normal, artinya belum ada image ditarik).

---

## 🔎 Troubleshooting

* Kalau error `failed to create shim task` → biasanya config cgroup belum diubah ke `systemd`.
* Kalau `containerd` tidak jalan setelah reboot → pastikan service sudah `enabled`.
* Untuk debugging lebih detail:

  ```bash
  journalctl -u containerd --no-pager
  ```

---

✅ Setelah Step 3 ini:

* Semua node sudah punya container runtime siap pakai.
* Cluster nanti bisa langsung pakai `kubeadm` tanpa error CRI.

---

# 🔹 Step 4 — Install Kubernetes Tools

## 1. Peran Setiap Komponen

* **kubeadm** → alat untuk *bootstrap* cluster Kubernetes (init/join).
* **kubelet** → agen di setiap node, bertugas menjalankan pod dan komunikasi dengan control plane.
* **kubectl** → CLI untuk interaksi dengan cluster (buat deployment, cek status, dll).

👉 Analogi untuk tim dengan background VM:

* `kubeadm` = seperti *installer VMware ESXi*.
* `kubelet` = agen di tiap host (seperti vSphere agent yang lapor ke vCenter).
* `kubectl` = vSphere Client/CLI untuk kontrol cluster.

---

## 2. Siapkan Repository Kubernetes

Jalankan di **semua node (master & worker)**.

Update sistem & install dependencies:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
```

Buat folder untuk kunci repository:

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
```

Import GPG key resmi Kubernetes:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

(📌 Sesuaikan versi `v1.30` dengan stable release terbaru — misalnya `v1.29` atau `v1.34` jika sudah rilis di official docs.)

Tambahkan repo Kubernetes:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

---

## 3. Install Kubernetes Packages

Update apt index:

```bash
sudo apt-get update
```

Install paket:

```bash
sudo apt-get install -y kubelet kubeadm kubectl
```

Cegah auto-upgrade (stabilitas penting):

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 4. Verifikasi Instalasi

Cek versi masing-masing:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

Kalau keluar versi (misal `v1.30.x`), artinya sukses ✅

---

## 🔎 Troubleshooting

* Jika `kubelet` gagal jalan setelah reboot → biasanya karena **swap belum dimatikan** (balik cek Step 1).
* Kalau `kubectl` error `config not found` → itu normal, karena kubeconfig baru dibuat setelah **init cluster** (Step 5).
* Kalau ada masalah repository GPG → cek apakah file `kubernetes-apt-keyring.gpg` benar-benar ada di `/etc/apt/keyrings/`.

---

✅ Setelah Step 4 ini:

* Semua node punya `kubeadm`, `kubelet`, `kubectl`.
* Cluster siap untuk **initialization** di master node.
---

# 🔹 Step 5 — Initialize Control Plane

## 1. Jalankan `kubeadm init`

Hanya di **master node**:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

### ✨ Penjelasan:

* `--pod-network-cidr=10.244.0.0/16` → ini CIDR default untuk **Flannel CNI**.
  Jika nanti pakai **Calico**, bisa pakai `192.168.0.0/16`.
* Saat command ini dijalankan:

  * `etcd` (database cluster) akan dibuat.
  * `kube-apiserver` (API utama cluster) aktif.
  * `kube-scheduler` dan `kube-controller-manager` jalan.
  * `kubelet` di master akan otomatis join.

Output akhirnya akan menampilkan sesuatu seperti:

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
...

Then you can join any number of worker nodes by running the following on each as root:

  kubeadm join 192.168.10.100:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

---

## 2. Konfigurasi `kubectl` di Master

Supaya bisa pakai `kubectl` tanpa root:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Tes koneksi cluster:

```bash
kubectl get nodes
```

Output awal biasanya hanya ada `master-node` dengan status `NotReady` (karena CNI belum diinstall → lanjut Step 6).

---

## 3. (Opsional) Izinkan Master Node Jalankan Workload

Secara default, master diberi **taint** supaya tidak dipakai untuk workload.
Kalau kamu ingin master juga bisa jalanin pod (karena resource terbatas di VM demo):

```bash
kubectl taint nodes master-node node-role.kubernetes.io/control-plane:NoSchedule-
```

👉 Analogi: taint itu kayak **policy di vSphere DRS** yang melarang VM ditempatkan di host tertentu. Kita bisa cabut kalau host mau ikut dipakai.

---

## 🔎 Troubleshooting

* **Error: swap is on** → balik ke Step 1, pastikan swap sudah off.
* **Error: ports 6443, 10250 already in use** → mungkin ada kubeadm init gagal sebelumnya → reset dengan:

  ```bash
  sudo kubeadm reset -f
  sudo systemctl restart kubelet
  ```
* **Master tetap NotReady** → normal sebelum install CNI (lanjut Step 6).

---

✅ Setelah Step 5 ini:

* Control plane aktif.
* Master node sudah terdaftar di cluster.
* Cluster menunggu CNI agar siap pakai.
---

# 🔹 Step 6 — Install Pod Network (CNI)

## 1. Konsep Dasar CNI

* **CNI (Container Network Interface)** = plugin yang mengatur jaringan antar-pod dan antar-node.
* Tanpa CNI → pod hanya “jalan” tapi tidak bisa komunikasi.
* Beberapa pilihan populer:

  * **Flannel** → ringan, cocok untuk lab/demo.
  * **Calico** → lebih lengkap (network policy, BGP), tapi lebih berat.

👉 Untuk simulasi ringan (seperti skenario 2–3 VM), **Flannel** lebih tepat.

---

## 2. Install Flannel

Jalankan di **master node**:

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Ini akan membuat:

* DaemonSet Flannel di semua node.
* Pod networking `10.244.0.0/16` (sesuai `--pod-network-cidr` di Step 5).

---

## 3. Verifikasi CNI

Cek apakah pods Flannel berjalan:

```bash
kubectl get pods -n kube-flannel
```

Harusnya status = `Running`.

Cek node status:

```bash
kubectl get nodes
```

Kalau berhasil, node master berubah dari `NotReady` → `Ready`.

---

## 4. Alternatif: Install Calico

Kalau ingin coba network policy lebih realistis:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

📌 Tapi ingat:

* `--pod-network-cidr` saat `kubeadm init` harus `192.168.0.0/16`.
* Resource usage lebih besar dibanding Flannel.

---

## 🔎 Troubleshooting

* **Node tetap NotReady** → biasanya karena:

  * `--pod-network-cidr` salah (misalnya pakai Calico tapi init pakai Flannel CIDR).
  * Modul kernel `br_netfilter` belum aktif (cek Step 2).
* **Flannel pod CrashLoopBackOff** → cek log:

  ```bash
  kubectl logs -n kube-flannel <nama-pod>
  ```
* **Ping antar pod gagal** → pastikan iptables aktif dan tidak diblok firewall VM host.

---

✅ Setelah Step 6 ini:

* Cluster sudah punya jaringan pod.
* Master node `Ready` → siap menerima workload.
* Worker node bisa join dengan lancar (Step 7).

---





