---
author: Maulana malik jabbar budianto
title: CI/CD With gitlab And Argocd at cluster kubernetes
description: CI/CD With gitlab And Argocd at cluster kubernetes
date: 2025-06-01T19:20:00+07:00
cover:
  image: /images/1_XV4tAJOiKrcEqNdwL2t_8A.webp
showToc: true
tocopen: false
tags:
  - kubernetes
---
CI/CD adalah singkatan dari Continuous Integration dan Continuous Deployment (atau Continuous Delivery). Ini adalah praktik dalam pengembangan perangkat lunak yang bertujuan untuk meningkatkan efisiensi, keandalan, dan kecepatan pengiriman perangkat lunak.

Continuous Integration (CI) adalah praktik di mana para pengembang secara teratur menggabungkan perubahan kode yang mereka lakukan ke dalam repositori utama proyek. Setiap kali ada perubahan kode yang di-commit, sistem CI akan secara otomatis membangun, menguji, dan memverifikasi perubahan tersebut. Ini memungkinkan tim pengembangan untuk mendeteksi kesalahan dan konflik lebih awal, mencegah masalah yang terkait dengan penggabungan kode yang dilakukan terlalu lama sebelum perilisan.

Continuous Deployment (atau Continuous Delivery) (CD) adalah langkah berikutnya setelah Continuous Integration. Setelah perubahan kode telah diuji dan diverifikasi melalui CI, mereka dapat secara otomatis diterapkan (dideploy) ke lingkungan produksi atau lingkungan lain yang relevan. Dengan CD, perubahan dapat diunggah dan dihadirkan kepada pengguna secara cepat dan berulang kali.

ArgoCD adalah alat open-source yang digunakan untuk pengiriman berkelanjutan (Continuous Deployment) dan manajemen konfigurasi aplikasi di lingkungan Kubernetes. ArgoCD berfungsi sebagai alat CI/CD khusus untuk aplikasi yang dideploy di lingkungan Kubernetes.

Itulah sedikit gambaran CI/CD di cluster kubernetes . Dua tools tersebut untuk sample kali ini CI nya menggunakan Gitlab dan CD nya menggunakan Argocd .

<p style="text-align: center"><img src="/images/1_XV4tAJOiKrcEqNdwL2t_8A.webp"></p>

# **Argocd Setup**

Saya menggunakan cluster kubernetes versi v1.26.3 dan menggunakan ingress-nginx dengan ingress.class bernama “nginx”

Buat terlebih dahulu namespace **argocd**

```
kubectl create namespace argocd
```

Setelah di buat namespace download manifest argocd

```
wget https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Edit manifest argocd , tambahkan opsi _— insecure_

```
nano install.yaml
```

```
#add line --insecure
containers:
      - args:
        - /usr/local/bin/argocd-server
        - --insecure 
```

Opsi “ — insecure” dalam konteks mengedit deployment ArgoCD menandakan bahwa koneksi ke server ArgoCD akan dilakukan tanpa memeriksa keaslian (insecure). Penggunaan opsi ini menonaktifkan verifikasi sertifikat SSL saat berkomunikasi dengan server ArgoCD.

Setelah manifest di edit deploy manifest tersebut

```
kubectl apply -n argocd -f install.yaml
```

Pastikan pod semua running tidak ada masalah , selanjut nya get password untuk keperluan login kedalam dashboard argocd .

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

> _Untuk username nya secara default argocd memakai user admin_

  
Setelah pod running tidak ada masalah deploy ingress untuk service argocd tersebut . Berikut manifest ingress untuk argocd

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-internal
  namespace: argocd
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.xass.site #Sesuaikan dengan host kalian
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

Deploy manifest ingress

```
kubectl apply -f ingress.yaml
```

Cek dashboard argocd untuk memastikan ingress berjalan dengan baik dan tidak ada masalah

<p style="text-align: center"><img src="/images/1_AXkQ8XbRDLf9TRqUcNtxTA.webp"></p>

# **Gitlab Repository/project Setup**

GitLab adalah sebuah platform pengembangan perangkat lunak berbasis web yang menyediakan berbagai fitur dan alat untuk mengelola siklus pengembangan perangkat lunak secara end-to-end. GitLab menyediakan solusi untuk manajemen repositori kode sumber, kolaborasi tim, pelacakan isu, otomasi CI/CD, pengujian otomatis, dan banyak lagi. Untuk membuat project/repository gitlab bisa singup/signin di [sini](https://gitlab.com/users/sign_in)

Membuat project , di sebelah kanan pilih new project untuk membuat project baru