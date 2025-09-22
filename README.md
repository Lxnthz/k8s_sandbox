# Panduan Lengkap (dalam Bahasa Indonesia) — Menyiapkan *Lightweight* Kubernetes (kubeadm + kubelet + kubectl) dengan **containerd**

> Target: 2 VM — **Master (Ubuntu Desktop)** yang juga boleh menjalankan workload, dan **Worker (Ubuntu Server)**.
> Tujuan: simulasi edukasional, mirip server on-premise, fokus pada penjelasan langkah-per-langkah untuk tim yang berpengalaman di virtualisasi monolitik.

> Sumber & rujukan utama: dokumentasi resmi Kubernetes & proyek terkait (dipakai sebagai dasar untuk langkah/langkah dan penjelasan). ([Kubernetes][1])

---

Saya susun panduan ini sesuai permintaan: tiap perintah/konfigurasi dijelaskan tujuannya; opsi alternatif & trade-offs; dan troubleshooting umum per fase. Gunakan sebagai materi presentasi — potong ke slide sesuai kebutuhan.

# 1. Prasyarat & Persiapan Awal

## 1.1 Hardware & Software Minimum (saran untuk demo 2-node)

* **Master (Ubuntu Desktop)**

  * CPU: 2 vCPU (lebih baik 4 vCPU untuk demo lebih lancar)
  * RAM: 4 GB minimum (8 GB direkomendasikan jika ingin jalankan UI + beberapa Pod)
  * Disk: 20 GB
  * OS: Ubuntu 22.04 / 24.04 (desktop environment optional)
* **Worker (Ubuntu Server)**

  * CPU: 1–2 vCPU
  * RAM: 2–4 GB
  * Disk: 10–20 GB
  * OS: Ubuntu Server 22.04 / 24.04

Catatan: angka di atas untuk tujuan edukasi/simulasi. Untuk produksi, setiap komponen kontrol-plane (atau node) butuh lebih banyak sumber daya dan HA. Pernyataan bahwa Kubernetes membutuhkan runtime CRI-compatible: Kubernetes sejak v1.24 menghapus dockershim — gunakan containerd atau CRI-O. ([Kubernetes][2])

## 1.2 Persiapan VM (di kedua node — master & worker)

Jalankan perintah berikut di kedua VM (sesuaikan user dengan sudo):

1. Update dan upgrade OS:

```bash
sudo apt update && sudo apt upgrade -y
```

**Mengapa:** memastikan paket terbaru (bugfix/security) sebelum instalasi komponen.

2. Matikan swap (Kubernetes memerlukan swap off — scheduler asumsikan memori yang konsisten):

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**Mengapa:** Kubernetes mengandalkan kubelet untuk manajemen memori; swap dapat menyebabkan perilaku scheduling tak terduga. (Jika Anda ingin eksperimen, ada cara untuk mengizinkan swap dengan konfigurasi khusus, tapi untuk tutorial ini kita nonaktifkan.) ([Kubernetes][1])

3. Set kernel params (untuk networking antar-pod; jalankan pada kedua node):

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

**Mengapa:** beberapa CNI plugin dan kube-proxy mengharapkan paket jaringan diteruskan melalui iptables. Tanpa ini, komunikasi pod-to-pod/per-node bisa gagal.

4. Konfigurasi IP statis (VM hypervisor/network):
   Untuk demonstrasi, tetapkan alamat statis pada tiap VM (mis. Master `192.168.56.10`, Worker `192.168.56.11`) atau buat DHCP reservation. Pastikan kedua VM saling ping.
   **Mengapa:** kubeadm/kubelet dan CNI mengandalkan konektivitas stabil antar node.

5. Matikan/atur firewall sementara (opsional untuk debugging):
   Jika ada `ufw` aktif:

```bash
sudo ufw disable
```

**Mengapa:** firewall dapat memblok port control-plane atau CNI; untuk demo lebih mudah matikan sementara — untuk produksi, buka port yang diperlukan secara eksplisit. (Nanti saya sertakan daftar port.)

---

# 2. Instalasi Container Runtime & Tools

## 2.1 Mengapa **containerd** (ringkasan, analogi dengan virtualisasi monolitik)

Bayangkan container runtime seperti hypervisor kecil yang khusus menjalankan *process containers*. `containerd` fokus pada lifecycle: image pull, storage, runtime exec. Kelebihannya:

* Lebih ringan daripada Docker (Docker bundling banyak tooling lain).
* Native CRI support (digunakan oleh Kubernetes setelah dockershim dihapus).
* Stabil dan didukung CNCF. ([Containerd][3])

### 2.1.1 Instalasi containerd di Ubuntu (langkah umum)

Di setiap node lakukan:

```bash
# 1. Instal dependensi
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# 2. Tambahkan repo (paket containerd dari distribution atau official)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y containerd.io
```

**Mengapa:** paket `containerd.io` adalah rilis containerd yang stabil.

### 2.1.2 Konfigurasi cgroup (penting untuk kubelet)

Kubernetes biasanya mengharapkan cgroup driver `systemd`. Generate config default dan set `SystemdCgroup = true`:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Edit /etc/containerd/config.toml:
# - Temukan bagian: [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
# - Pastikan: SystemdCgroup = true
# Alternatif: sed one-liner (verifikasi file setelahnya)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Mengapa:** konsistensi cgroup antara container runtime dan kubelet mencegah error resource/cgroup mismatch. Banyak panduan resmi & komunitas merekomendasikan ini. ([Fortaspen - Cloud Guides and Tech Tips][4])

**Troubleshooting containerd:**

* `sudo journalctl -u containerd -f` untuk melihat log startup.
* Jika image pull gagal, cek DNS/HTTP proxy dan MTU (beberapa jaringan virtual menuntut MTU lebih kecil).

## 2.2 Instalasi kubeadm, kubelet, kubectl

Di kedua node (master & worker):

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] \
https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
# ganti versi sesuai kebutuhan; contoh: 1.28.0 (pilih versi stabil yang Anda inginkan)
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**Mengapa tiap alat:**

* **kubeadm**: tool untuk *bootstrap* cluster (membuat control plane, buat token join, dsb.). ([Kubernetes][1])
* **kubelet**: agen node — bertanggung jawab menjalankan Pods berdasarkan spec yang diberikan API server.
* **kubectl**: CLI client untuk berinteraksi dengan API server (deploy, inspect, debug).

**Catatan versi:** gunakan versi kubeadm yang kompatibel dengan versi kubelet/kubectl; dokumentasi kubeadm menyarankan pasangan versi ±1 minor. (Untuk demo, gunakan versi distribusi apt terbaru kecuali Anda punya kebutuhan versi tertentu.)

---

# 3. Inisialisasi Cluster pada Master

> Jalankan langkah berikut **hanya pada Master node**.

## 3.1 Buat konfigurasi minimal atau gunakan command line

Pilihan cepat (command line) — untuk demo dua node dengan CNI yang membutuhkan `pod-network-cidr` (mis. Calico dengan default membutuhkan 192.168.0.0/16 kadang, Flannel 10.244.0.0/16). Sering dipakai Flannel `10.244.0.0/16` atau Calico juga bisa tanpa CIDR terspesifikasi (tapi beberapa opsi memerlukan). Untuk konsistensi, kita tunjukkan contoh dengan `--pod-network-cidr=192.168.0.0/16`:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

**Penjelasan flag utama:**

* `--pod-network-cidr`: menginformasikan kube-controller-manager tentang rentang IP yang akan dipakai untuk Pod IP. Beberapa CNI plugin memerlukan Anda menentukan CIDR di `kubeadm init` agar controllermanager menetapkan `--cluster-cidr`. Jika Anda menggunakan plugin yang tidak butuh, flag ini bisa diabaikan — tetapi pastikan plugin & kubeadm konfigurasi cocok. ([Kubernetes][5])

**Output penting:** kubeadm akan menampilkan:

* instruksi untuk membuat file kubeconfig (untuk kubectl),
* *join command* (untuk worker), dan
* (opsional) certificate-key jika menggunakan multi control plane.

Simpan nilai *join command* atau copy ke tempat aman (atau Anda bisa regenerate token nanti).

## 3.2 Mengerti control-plane components (analogi dengan virtualisasi)

* **etcd**: *distributed key-value store* — state cluster (seperti metadata database VM manager). Jika analogi virtualisasi: ini seperti DB pusat yang menyimpan konfigurasi VM & metadata.
* **kube-apiserver**: pintu gerbang (API) — semua perubahan cluster lewat sini. Analogi: management API hypervisor/cloud controller.
* **kube-controller-manager**: menjalankan kontrol loop (node controller, replication controller, dsb.). Analogi: proses manajemen tugas rutin di hypervisor.
* **kube-scheduler**: menempatkan pod ke node (mirip scheduler penempatan VM ke host).

Kubeadm men-deploy static pods untuk control-plane di `/etc/kubernetes/manifests` sehingga kubelet akan mengelolanya sebagai proses static Pod.

## 3.3 Konfigurasi kubectl (agar user biasa bisa gunakan kubectl)

Untuk user non-root (misal user Anda):

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Mengapa:** `kubeadm init` buat file admin kubeconfig di `/etc/kubernetes/admin.conf` — kita copy agar `kubectl` berkomunikasi dengan API server dengan kredensial admin.

## 3.4 Untainting master supaya bisa schedule workload (opsional untuk demo)

Secara default master diberi taint `node-role.kubernetes.io/control-plane:NoSchedule` sehingga tidak menjalankan workload biasa. Untuk memperbolehkan master menjalankan Pod (berguna untuk demo 2-node):

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
# atau untuk taint dengan key yang sedikit berbeda (versi lama)
kubectl taint nodes --all node-role.kubernetes.io/master-
```

**Penjelasan:** `taint` mencegah scheduler menempatkan pod di node kecuali Pod memiliki toleration yang sesuai. Menghapus taint memudahkan demo tapi untuk produksi biasanya **tidak** disarankan kecuali Anda tahu dampaknya.

---

# 4. Networking & Menambahkan Worker

## 4.1 Konsep CNI (Container Network Interface)

CNI adalah standar plugin jaringan untuk Kubernetes agar setiap Pod bisa komunikasi (layer 3). Tanpa CNI, Pods tidak akan saling terhubung meskipun node sudah terdaftar. CNI bertanggung jawab memberikan IP, routing antar node, dan (opsional) network policy enforcement. ([Kubernetes][1])

## 4.2 Memilih CNI plugin — **Calico** (rekomendasi untuk demo)

**Mengapa Calico:** fitur lengkap (policy, BGP opsional, performa baik), dokumentasi matang. Untuk demo kecil, gunakan manifest quickstart. Alternatif: **Flannel** (sangat sederhana — overlay VXLAN, cocok jika hanya butuh L3 basic), atau **Weave Net**. Trade-offs ringkas:

* **Calico**: fitur network policy + routing; sedikit lebih kompleks.
* **Flannel**: sederhana, mudah install, tapi tidak sediakan policy canggih.
* **Cilium**: modern, eBPF based — powerfull tapi lebih kompleks (bagus untuk performance). ([Calico Documentation][6])

### 4.2.1 Install Calico (contoh - jalankan pada master setelah `kubeadm init` selesai dan `kubectl` dikonfigurasi)

```bash
# contoh mengambil manifest Calico (versi dapat berubah; gunakan dokumentasi resmi untuk versi terbaru)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml
```

**Penjelasan:** `kubectl apply -f <manifest>` membuat resource Kubernetes (DaemonSet, CRDs, dsb.) yang menjalankan komponen Calico pada cluster. Perhatikan versi URL; gunakan link yang direkomendasikan di dokumentasi Calico untuk versi saat ini. ([Calico Documentation][6])

**Troubleshooting CNI:**

* `kubectl get pods -n kube-system` untuk memeriksa `calico-node`, `kube-proxy`, dsb. Pastikan status `Running`.
* Jika Pod CNI crash/CrashLoopBackOff, cek log: `kubectl logs -n kube-system <pod-name> -c calico-node`.
* Pastikan `--pod-network-cidr` yang dipakai saat `kubeadm init` cocok dengan dokumentasi CNI (beberapa plugin mengharuskan CIDR tertentu).

## 4.3 Generate & Jalankan join command di Worker

Setelah kubeadm init, output akan menampilkan `kubeadm join ... --token ... --discovery-token-ca-cert-hash sha256:...`. Jika hilang, Anda bisa buat token baru di master:

```bash
# buat token baru (di Master)
sudo kubeadm token create --print-join-command
```

Perintah di atas akan mengeluarkan seluruh `kubeadm join` command, mis:

```bash
sudo kubeadm join 192.168.56.10:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**Jalankan perintah join itu di Worker (sebagai root):**

```bash
sudo kubeadm join 192.168.56.10:6443 --token ... --discovery-token-ca-cert-hash ...
```

**Penjelasan:**

* Token adalah mekanisme bootstrap sementara yang mengautentikasi worker ke control plane.
* `discovery-token-ca-cert-hash` memastikan Worker memverifikasi server API yang benar (mitigasi MITM).
* Jika token expired, buat yang baru. kubeadm juga mendukung metode lain (bootstrap token + CA cert, dsb).

**Verifikasi pada Master:**

```bash
kubectl get nodes
```

Anda seharusnya melihat `master` dan `worker` dengan status `Ready` setelah beberapa saat (tergantung CNI siap).

---

# 5. Verifikasi & Deploy Pertama

## 5.1 Periksa kesehatan cluster & sistem pods

Di Master:

```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl get cs   # componentstatuses (deprecated, gunakan kubectl get apiservices / healthz endpoints jika perlu)
```

**Checklist:**

* `kubectl get nodes` -> kedua node `Ready`.
* `kubectl get pods -n kube-system` -> `coredns`, `kube-proxy`, `calico-node` (atau CNI setara) `Running`.
  Jika ada yang tidak Ready: periksa logs `kubectl logs -n kube-system <pod>`, sistemd logs untuk containerd, dan network settings.

## 5.2 Contoh Deploy sederhana: NGINX Deployment + Service

Buat file `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx-demo
  type: NodePort   # untuk demo, mudah diakses dari luar node
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

Jalankan:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployments
kubectl get pods -l app=nginx-demo
kubectl get svc nginx-svc
```

**Penjelasan konsep:**

* **Deployment:** mengatur jumlah replika Pod, strategi rolling update, dan menjaga desired state (seperti autoscaler kecil). Analogi: *Deployment* mirip "golden image + desired instances" di virtualisasi, tetapi jauh lebih ringan karena Pod berbagi kernel host.
* **Service:** abstraksi jaringan untuk mengakses Pod. `NodePort` membuka port di tiap node (contoh `30080`) sehingga Anda bisa mengakses `http://<node-ip>:30080`. Untuk produksi gunakan `LoadBalancer` (cloud) atau Ingress + Service.

## 5.3 Akses aplikasi dari luar cluster

Jika Anda gunakan `NodePort` 30080:

* Akses `http://<master-ip>:30080` (atau worker IP) dari host lokal/hypervisor network.
  Jika firewall aktif, buka port `30080`.

Alternatif (lebih elegan untuk demo lokal): gunakan `kubectl port-forward` pada master:

```bash
kubectl port-forward svc/nginx-svc 8080:80
# browse http://localhost:8080
```

**Penjelasan:** `port-forward` membuat tunnel sementara dari lokal ke service/pod — berguna untuk debugging.

---

# 6. Alternatif & Trade-offs (ringkasan cepat)

* **Runtime: containerd vs CRI-O vs Docker Shim**

  * `containerd`: simple, stabil, didukung luas. ([Containerd][3])
  * `CRI-O`: ringan dan khusus untuk Kubernetes, pilihan bagus untuk security-focused env.
  * `Docker (dockershim)` sudah dihapus dari kode sumber; tetap bisa pakai Docker Engine dengan plugin CRI adapter tapi itu menambah lapisan tambahan — tidak dianjurkan untuk setup modern. ([Kubernetes][2])

* **CNI: Calico vs Flannel vs Cilium**

  * Calico: fitur policy & routing, sedikit kompleks. ([Calico Documentation][6])
  * Flannel: sederhana, cepat deploy, cocok demo.
  * Cilium: eBPF power, bagus untuk performa dan observability, tapi kurva belajar lebih tinggi.

* **Master scheduling** — untuk demo kecil, untaint master; untuk produksi, jangan untaint (isolasi control plane).

---

# 7. Daftar Port Penting (untuk network/firewall)

* `6443` — Kubernetes API Server (control plane).
* `2379-2380` — etcd server client API.
* `10250` — kubelet API (node).
* `30000–32767` — NodePort default range.
  (lihat dokumentasi resmi untuk daftar lengkap & port tambahan seperti BGP/Calico jika digunakan).

---

# 8. Troubleshooting Umum & Tips

1. **Node stuck in `NotReady`**

   * Pastikan containerd berjalan (`systemctl status containerd`).
   * `kubectl describe node <node>` untuk events (sering karena CNI belum terpasang).
   * Periksa `kubelet` logs: `sudo journalctl -u kubelet -f`.

2. **Pods CrashLoopBackOff**

   * `kubectl logs <pod>` dan `kubectl describe pod <pod>` untuk detail.
   * Cek image pull error (registry, network, credentials).
   * Resource limits mungkin terlalu kecil -> pod OOMKilled.

3. **CNI Pods Crash atau Tidak Ready**

   * Periksa `kubectl get pods -n kube-system`.
   * Pastikan `sysctl` bridge rules sudah diterapkan.
   * Cek versi manifest CNI & apakah cocok dengan `pod-network-cidr`.

4. **Token expired untuk join**

   * Buat token baru: `kubeadm token create --print-join-command` di master.

5. **Masalah cgroup**

   * Jika kubelet melaporkan masalah cgroup, pastikan `SystemdCgroup = true` di `/etc/containerd/config.toml` dan restart containerd & kubelet.

6. **Time sync / NTP**

   * Pastikan waktu disinkronkan (ntp/chrony) agar certificate expiry/discovery tidak bermasalah.

---

# 9. Skenario Demo / Alur Presentasi (saran)

1. Perkenalan konsep (API server, etcd, kubelet) — gunakan analogi VM manager/DB.
2. Tunjukkan master init (`kubeadm init`) & jelaskan output.
3. Pasang CNI (Calico) & jelaskan kenapa CNI perlu dipasang.
4. Jalankan `kubeadm join` di worker, tunjukkan `kubectl get nodes`.
5. Deploy NGINX, buka di browser (NodePort atau port-forward).
6. Demonstrasi rolling update: ubah image versi, `kubectl apply` dan `kubectl rollout status deployment/nginx-deploy`.
7. Tutup dengan troubleshooting demo (contoh: hidupkan kembali swap untuk tunjukkan error, lalu perbaiki).

---

# 10. Referensi Utama (dok. resmi)

* Kubeadm init & create cluster docs — Kubernetes official. ([Kubernetes][5])
* Container runtimes & containerd docs. ([Kubernetes][2])
* Calico quickstart & on-prem docs. ([Calico Documentation][6])

---

## Lampiran: Perintah cepat untuk checklist (copy-paste)

Di **kedua node**:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

Di **kedua node** (install kubeadm/kubelet/kubectl):

```bash
# (lihat bagian 2.2)
```

Di **master**:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml
```

Di **worker**:

```bash
# jalankan kubeadm join command yang diberikan oleh kubeadm init atau dari `kubeadm token create --print-join-command`
```

---

[1]: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/?utm_source=chatgpt.com "Creating a cluster with kubeadm - Kubernetes"
[2]: https://kubernetes.io/docs/setup/production-environment/container-runtimes/?utm_source=chatgpt.com "Container Runtimes - Kubernetes"
[3]: https://containerd.io/?utm_source=chatgpt.com "containerd – An industry-standard container runtime with an ..."
[4]: https://www.fortaspen.com/install-kubernetes-containerd-ubuntu-linux-22-04/?utm_source=chatgpt.com "Install Kubernetes / Containerd (Ubuntu Linux 22.04) - Fortaspen"
[5]: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/?utm_source=chatgpt.com "kubeadm init | Kubernetes"
[6]: https://docs.projectcalico.org/getting-started/kubernetes/quickstart?utm_source=chatgpt.com "Calico quickstart guide"
