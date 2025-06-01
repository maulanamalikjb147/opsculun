---
author: Maulana malik jabbar budianto
title: Create NFS Server at cluster Kubernetes
description: NFS Server for storage class kubernetes
date: 2025-06-01T19:20:00+07:00
cover:
  image: /images/0_kboKokNGf42PhyOs.webp
  hiddenInList: true
showToc: true
tocopen: false
tags:
  - kubernetes
---
NFS Server adalah komponen perangkat lunak atau layanan yang memungkinkan untuk membagikan sistem file melalui protokol NFS (Network File System). Sebuah NFS Server menyediakan akses jaringan ke sistem file yang disimpan di server kepada klien yang terhubung melalui jaringan.

Dengan menggunakan NFS Server, klien yang terhubung ke jaringan dapat memanfaatkan sistem file yang di-host oleh server NFS seolah-olah mereka berada pada sistem lokal mereka sendiri. Ini memungkinkan untuk berbagi data dan sistem file secara efisien di antara beberapa mesin atau perangkat yang terhubung dalam jaringan.

NFS Server menyediakan mekanisme ekspor file, di mana administrator server dapat menentukan direktori mana yang akan dibagikan dan kepada klien mana aksesnya diizinkan. Klien dapat melakukan mounting (memasang) direktori yang dibagikan dari server NFS ke dalam sistem file mereka sendiri, sehingga dapat mengakses, membaca, menulis, dan melakukan operasi lainnya pada file dan direktori tersebut seolah-olah mereka berada pada sistem lokal.

Dengan menggunakan NFS, pengguna dapat memanfaatkan manfaat bersama dari penyimpanan dan pengelolaan data terpusat, serta mengizinkan akses file yang efisien dan terstruktur melalui jaringan.

NFS di dalam cluster Kubernetes merujuk pada penggunaan server NFS (Network File System) untuk menyediakan persistent volume (PV) dan persistent volume claim (PVC) kepada aplikasi yang berjalan di dalam cluster. Dalam konteks ini, server NFS digunakan sebagai penyimpanan bersama yang dapat diakses oleh banyak pod di dalam cluster.

Dalam Kubernetes, penggunaan NFS sebagai penyimpanan persisten memungkinkan aplikasi untuk menyimpan dan membagikan data di antara pod-pod yang berjalan di dalam cluster. Beberapa keuntungan penggunaan NFS di cluster Kubernetes termasuk:

1.  Penyimpanan Bersama: Dengan menggunakan server NFS, beberapa pod dapat mengakses dan membagikan data yang disimpan di dalam sistem file yang sama.
    
2.  Persistensi: Data yang disimpan di NFS akan bertahan bahkan jika pod atau wadah yang berjalan di dalam cluster dihentikan atau dimulai kembali. Ini memungkinkan aplikasi untuk mempertahankan data mereka dalam skenario tersebut.
    
3.  Skalabilitas: Server NFS dapat dikonfigurasi untuk menangani sejumlah besar permintaan dari pod-pod yang berjalan di dalam cluster Kubernetes, sehingga memungkinkan skala aplikasi yang lebih besar.
    

Untuk menggunakan NFS di dalam cluster Kubernetes, Anda perlu mengonfigurasi objek StorageClass untuk menunjuk ke server NFS sebagai penyedia penyimpanan, dan kemudian membuat persistent volume claim (PVC) yang mengikat ke PV dengan tipe NFS. PV dan PVC ini dapat digunakan oleh pod-pod di dalam cluster untuk menyimpan dan mengakses data yang persisten.

Penggunaan NFS di cluster Kubernetes dapat memberikan fleksibilitas dan kemudahan dalam mengelola penyimpanan persisten bagi aplikasi yang berjalan di dalam cluster. Berikut sedikit pembahasan NFS Server dan Fungsi NFS Server di cluster kubernetes.

Berikut topologi NFS provisioning untuk cluster kubernetes

<p style="text-align: center"><img src="/images/0_kboKokNGf42PhyOs.webp"></p>

# **NFS Setup**

My Environment

| Hostname | IP  |
| --- | --- |
| k8s-loadbalancer1 | 10.10.10.9 |
| k8s-loadbalancer2 | 10.10.10.10 |
| k8s-master1 | 10.10.10.11 |
| k8s-master2 | 10.10.10.12 |
| k8s-master3 | 10.10.10.13 |
| k8s-worker1 | 10.10.10.14 |
| k8s-worker2 | 10.10.10.15 |
| k8s-worker3 | 10.10.10.16 |
| k8s-vrrp | 10.10.10.50 |

> _Disini saya akan membuat skenario master1 dijadikan sebagai NFS Server_

## **Install OS packages**

Install di **node** yang mau di jadikan **NFS**

```
sudo apt-get update
sudo apt-get install nfs-common nfs-kernel-server -y
```

Buat directory untuk di pakai penyimpanan sebagai NFS Server

```
sudo mkdir -p /data/nfs1
sudo chown nobody:nogroup /data/nfs1
sudo chmod g+rwxs /data/nfs1
```

Export Directory

```
echo -e "/data/nfs1\t10.10.10.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -av
```

> _Sesuaikan IP dengan Subnet kalian_

Setelah directory di export restart NFS Server tersebut

```
sudo systemctl restart nfs-kernel-server
sudo systemctl status nfs-kernel-server
```

Cek hasil export tadi

```
/sbin/showmount -e localhost
/sbin/showmount -e 10.10.10.11
```

Setelah setup nfs-server sudah di lakukan . Install NFS Client di setiap **worker node**

```
sudo apt update
sudo apt install nfs-common -y
```

Cek hasil export tadi di worker node

```
/sbin/showmount -e 10.10.10.11
```

Install deployment NFS menggunakan helm , apabila tidak punya helm3 silahkan install terlebih dahulu

**Jalankan di Node Master**

```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

Tambah repo ke helm

```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
```

Install chart nfs yang telah di tambahkan repo nya tadi

> _Silahkan sesuaikan dengan info cluster masing — masing punya kalian_

```
helm install nfs-subdir-external-provisioner \
nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=10.10.10.11 \
--set nfs.path=/data/nfs1 \
--set storageClass.onDelete=true 
```

Setelah pod running jadikan Storage Class tersebut sebagai Storage Class default

```
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Untuk cek nama Storage class bisa cek dengan command

```
kubectl get sc
```

Setelah set default storage class , berikut output apabila action tersebut berhasil di lakukan

<p style="text-align: center"><img src="/images/1_fE6m6YxU1MIqLX6SQ2f_AQ.webp"></p>

Untuk pengujian Storage class dan NFS server itu berjalan . Buat lah pvc dan storage class nya mengarah ke storage class yang telah di buat

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sc-nfs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 2Gi
```

Deploy PVC tersebut

```
kubectl apply -f test-pvc.yaml
```

Check apakah pvc itu telah bound ke storage class atau belum

```
kubectl get pvc sc-nfs-pvc
```

<p style="text-align: center"><img src="/images/1_PDgPeQNtSDr3eyM-zerTKg.webp"></p>

Validasi hasil pvc tersebut dengan deploy sebuah pod nginx dan di arahkan ke pvc yang sudah di buat tadi

```
apiVersion: apps/v1 
kind: Deployment
metadata:
  labels:
    app: sc-nginx
  name: sc-nfs-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sc-nginx
  template:
    metadata:
      labels:
        app: sc-nginx
    spec:
      volumes:
      - name: nfs-test
        persistentVolumeClaim:
          claimName: sc-nfs-pvc
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nfs-test # template.spec.volumes[].name
          mountPath: /usr/share/nginx/html # mount inside of container
          #readOnly: true # if enforcing read-only on volume
```

Setelah apply manifest tersebut , lakukan step ini untuk provisioning pod dengan pvc tadi

```
# check pod status
kubectl get pods -l=app=sc-nginx

# capture unique pod name
pod_name=$(kubectl get pods -l=app=sc-nginx --no-headers -o=custom-columns=NAME:.metadata.name)

# capture unique pod name
pod_name=$(kubectl get pods -l=app=sc-nginx --no-headers -o=custom-columns=NAME:.metadata.name)

# create html file on share
kubectl exec -it $pod_name -- sh -c "echo \"<h1>hello test PVC yang mengarah ke Storage class</h1>\" > /usr/share/nginx/html/hello.html"

# pull file using HTTP
kubectl exec -it $pod_name -- curl http://localhost:80/hello.html
<h1>hello test PVC yang mengarah ke Storage class</h1>
```

![](/images/1_J1xX1jq8VtqEMXaLzqZDuw.webp)

Voilaaa , provisioning NFS Server di Cluster Kubernetes telah berhasil. Apabila anda mau cek hasil yang tadi dibuat , bisa di lihat data yang ada di pvc tadi di **instance** yang di install **nfs server**

```
ls -l /data/nfs1
```

<p style="text-align: center"><img src="/images/1_9SrkNA6nEbxE6ylYcCNu_g.webp"></p>

Dan apabila anda melihat hasil pvc nginx tadi , maka output nya akan sama seperti test curl ke pod tersebut

```
cat /data/nfs1/default-sc-nfs-pvc*/hello.html
```

<p style="text-align: center"><img src="/images/1_8ZJMYijwQrXYSkVr2rU_SQ.webp"></p>

Mohon maaf apabila ada kalimat yang susah dipaham / kurang mengerti / terlalu berbelit2 , sekian ges chersss.