---
author: Maulana malik jabbar budianto
title: MetalLB Kubernetes
description: Metallb for kubernetes
date: 2025-06-02T14:54:00+07:00
cover:
  image: /images/0_I2nsvDg__tgUSC-n (1).webp
showToc: true
tocopen: false
tags:
  - kubernetes
---
Apa itu metallb?

MetalLB adalah sebuah implementasi layanan LoadBalancer untuk lingkungan Kubernetes yang berjalan di infrastruktur bare metal atau virtualized. Biasanya, layanan LoadBalancer di Kubernetes secara default tersedia hanya dalam lingkungan cloud publik yang mendukung integrasi khusus untuk memprovisioning load balancer eksternal.

Namun, ketika menggunakan Kubernetes di lingkungan bare metal atau virtualized, tidak ada layanan LoadBalancer yang otomatis tersedia. Inilah mengapa MetalLB hadir sebagai solusi alternatif. MetalLB memungkinkan Anda untuk menggunakan IP LoadBalancer di lingkungan infrastruktur Anda sendiri.

MetalLB mengimplementasikan protokol Kubernetes Service LoadBalancer agar kompatibel dengan lingkungan infrastruktur bare metal atau virtualized. Dengan MetalLB, Anda dapat mengkonfigurasi alamat IP LoadBalancer dan mengarahkannya ke backend pod yang sesuai.

Proses kerja MetalLB melibatkan penggunaan protokol ARP (Address Resolution Protocol) atau protokol BGP (Border Gateway Protocol) untuk mengiklankan alamat IP LoadBalancer yang tersedia ke jaringan Anda. Ketika permintaan masuk ke alamat IP LoadBalancer, MetalLB akan meneruskan permintaan ke backend pod yang sesuai dengan metode load balancing yang telah dikonfigurasi.

MetalLB dapat diinstal sebagai tambahan ke kluster Kubernetes Anda dan dikonfigurasi menggunakan objek Kubernetes seperti ConfigMap dan Service. Anda dapat mengatur pengaturan seperti rentang alamat IP yang tersedia, metode load balancing yang digunakan, dan banyak lagi.

Dengan MetalLB, Anda dapat mengaktifkan fungsi LoadBalancer di lingkungan bare metal atau virtualized Anda dan mengelola lalu lintas aplikasi dengan mudah di Kubernetes.

Dari kesimpulan di atas MetalLB = Loadbalancer ? terus apa bedanya dengan Nginx / Haproxy ? . Kurang lebih seperti ini

1.  Metallb :  
    \- MetalLB adalah solusi layanan LoadBalancer khusus untuk Kubernetes yang dirancang untuk lingkungan bare metal atau virtualized.  
    \- MetalLB tidak memberikan fitur load balancing tingkat aplikasi seperti HAProxy atau Nginx. Fungsinya lebih fokus pada pengaturan lalu lintas ke backend pod di tingkat kluster Kubernetes.  
    \- MetalLB bekerja sebagai komponen di dalam kluster Kubernetes dan mengimplementasikan protokol Kubernetes Service LoadBalancer agar kompatibel dengan lingkungan infrastruktur yang tidak memiliki integrasi LoadBalancer bawaan.
    
2.  HAProxy :  
    \- HAProxy adalah solusi load balancing open-source yang dapat digunakan di lingkungan Kubernetes maupun di luar Kubernetes.  
    \- HAProxy berfungsi sebagai reverse proxy dan dapat menyeimbangkan lalu lintas HTTP dan TCP di tingkat aplikasi.  
    \- HAProxy menyediakan berbagai algoritma load balancing yang canggih, seperti round-robin, least connections, dan weighted round-robin.
    
3.  Nginx :  
    \- Nginx juga merupakan solusi reverse proxy dan load balancing yang populer dan dapat digunakan baik di lingkungan Kubernetes maupun di luar Kubernetes.  
    \- Nginx menyediakan fitur load balancing tingkat aplikasi dan mendukung berbagai algoritma load balancing, seperti round-robin, least connections, dan IP hash.  
    \- Nginx memiliki kemampuan yang kuat dalam mengelola permintaan HTTP, memproses SSL, dan mengoptimalkan lalu lintas HTTP.
    

Mungkin yang saya dapat sampaikan untuk perbedaan nya kurang lebih seperti itu 😁 , selebih nya bisa di explore lagi .

# **Setup Metallb**

<p style="text-align: center"><img src="/images/1_QdghPVDddEn9YMRMR__QHw.webp"></p>

Install MetalLb

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

<p style="text-align: center"><img src="/images/1_5YcrgGl8Upt5NlcOT1O7TA.webp"></p>

Setelah manifest di apply terlihat manifest tersebut mendeploy Deployment **controller** dan Daemonset **speaker** . Setelah manifest terdeploy kita buat ippool untuk service yang membutuh kan ippool dari MetalLb

```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.10.10.100-10.10.10.110
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-pool-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
```

Karena subnet cluster saya memakai 10.10.10.0/24 jadi saya memakai 10.10.10.100–10.10.10.110 untuk ippool tersebut. Setelah Ippool telah di apply . cek untuk ippool tersebut

```
kubectl get ipaddresspools.metallb.io -n metallb-system
```

<p style="text-align: center"><img src="/images/1_IL_CI2JVi0RFYpht7JAj7A.webp"></p>

Untuk pengetesan tersebut kita deploy nginx dengan type service LoadBalancer

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

<p style="text-align: center"><img src="/images/1_WF-Tf5-F_3Y-ShlAkLJJ9w.webp"></p>

Bisa di lihat , Deployment nginx mendapat kan ip External-Ip dari MetalLb dimana ip tersebut masuk kedalam ippool Metallb. Untuk pengetesan bisa dicoba dengan meng curl ip external tersebut di dalam cluster dan di luar cluster.untuk mengetesan curl ip external ip tersebut.

Voilaaa Metallb telah berhasil di deploy dan di aplikasikan , mungkin sample sederhana itu yang bisa saya share , Mohon maaf apabila ada kalimat yang susah dipaham / kurang mengerti / terlalu berbelit2 , sekian ges chersss. ☕