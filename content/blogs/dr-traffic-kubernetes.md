---
author: Maulana malik jabbar budianto
title: DR Traffic Kubernetes
description: DR Kubernetes
date: 2025-06-02T20:02:00+07:00
showToc: true
tocopen: false
tags:
  - kubernetes
---
Disaster Recovery (DR) adalah serangkaian proses, kebijakan, dan teknologi yang dirancang untuk mengatasi atau memulihkan sistem IT setelah terjadinya trouble serius yang mengancam operasional normal. Trouble yang dimaksud bisa berupa hal-hal seperti kegagalan perangkat keras, serangan siber, Human error, atau situasi tak terduga lainnya yang dapat menyebabkan gangguan atau kerusakan pada sistem dan data.

Mungkin penjelasan dari mbah google tentang DR begitu yang intinya ketika cluster Utama mengalami down atau terjadi gangguan , trafik atau request bisa di arahkan ke cluster DR atau Backup . Berikut untuk topologi nya :

![](https://miro.medium.com/v2/resize:fit:700/1*a8o1XeUTVmYTxhbkfUWWNA.png)

> Nginx ini bisa di ganti dengan f5 lebih bagusnya

# **Setup DR**

Untuk setup DR yang pertama di butuhkan Service openvpn di mana service tersebut agar Cluster dari GCP dan di cluster Hetzner bisa satu koneksi karena mendapat ip internal dari Openvpn tersebut . Untuk cara setup nya silahkan cari di google karena kalo di share disini kepanjangan , atau lain kali saya share juga .

## **Setup Nginx Reverse Proxy at AWS**

Install nginx

```
sudo apt install nginx 
```

Backup Konfigurasi lama

```
sudo mv /etc/nginx/{nginx.conf,nginx.conf.orig}
```

Setup Konfigurasi baru

```
sudo nano /etc/nginx/nginx.conf
```

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
     worker_connections 768;
     # multi_accept on;
}

http {
     # include /etc/nginx/conf.d/*.conf;
     # include /etc/nginx/sites-enabled/*;
}

stream {
    include /etc/nginx/stream.d/*.conf;

    log_format basic '$remote_addr [$time_local] '
                 '$protocol $status $bytes_sent $bytes_received '
                 '$session_time "$upstream_addr" '
                 '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';

    access_log /var/log/nginx/access.log basic;
    error_log  /var/log/nginx/error.log;
}
```

Buat direktori stream.d

```
sudo mkdir /etc/nginx/stream.d
```

Buat konfigurasi Reverse Proxy , dengan backend ke haproxy dari cluster GCP dan Hetzner

```
nano /etc/nginx/stream.d/reverse.conf 
```

```
upstream k8s-ingress-http {
    server 10.8.0.3:80;
    server 10.8.0.2:80 backup;
}

server {
    listen 80 ;
    proxy_pass k8s-ingress-http;
    proxy_connect_timeout 10s;
    proxy_timeout 60s;
}
```

Konfigurasi tersebut , ketika server yang utama itu mati otomatis trafik di alirkan semua ke server yang kedua , Apabila server yang utama nyala lagi otomatis semua trafik akan menuju ke server utama.

Pastikan konfigurasi nginx benar

```
sudo nginx -t
```

Apabila sudah benar tidak ada error , restart nginx

```
systemctl restart nginx.service
```

Untuk setup domain dengan instance AWS , kalian harus punya domain yang sudah ter integrasi dengan cloudflare , dengan setup seperti ini

<p style="text-align: center"><img src="https://miro.medium.com/v2/resize:fit:700/1*aBLpgbB-cHNW4GgoEjD1XQ.png"></p>

## **Setup Reverse proxy with Haproxy**

Install Haproxy

```
sudo apt install haproxy
```

Backup konfigurasi default

```
sudo cp /etc/haproxy/haproxy.cfg haproxy.cfg.bak
```

Setup konfigurasi yang baru

```
nano /etc/haproxy/haproxy.cfg
```

Tambahkan konfigurasi ini di paling bawah

```
frontend http-in
    bind *:80
    default_backend backend_servers

backend backend_servers
    balance roundrobin
    server maul-worker1 10.30.10.101:80 check
    server maul-worker2 10.30.10.102:80 check
```

> _Karena saya menggunakan ingress type hostport jadi default port nya 80 , dan ingress saya bertype daemonset . Sesuaikan dengan cluster kalian_

Check konfigurasi yang baru

```
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

Apabila konfigurasi telah benar dan tidak ada error , restart haproxy tersebut

```
sudo systemctl restart haproxy.service
```

Setelah konfigurasi semua telah disetup mari kita test

## Sanity DR Traffic

Untuk pengetesan nya saya hanya memakai deployment simple nginx dengan custom index menggunakan configmap , berikut untuk sample nya

```
kind: Deployment
metadata:
  name: deployment1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - name: apache
        image: httpd:2.4
        ports:
        - containerPort: 80
        volumeMounts:
        - name: custom-index
          mountPath: /usr/local/apache2/htdocs/index.html
          subPath: index.html
      volumes:
      - name: custom-index
        configMap:
          name: apache-config
---
apiVersion: v1
kind: Service
metadata:
  name: apache-service
spec:
  selector:
    app: apache
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: apache-config
data:
  index.html: |
    <html>
      <head>
        <title>TEST DR</title>
      </head>
      <body>
        <center>
         <h1> TEST DR CUYYY </h1>
        </center>
      </body>
    </html>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-dr
spec:
  ingressClassName: nginx
  rules:
    - host: testdr.xass.site
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: apache-service
                port:
                  number: 80
```

> _Apabila mau mencoba , Silahkan sesuaikan dengan cluster kalian_

Untuk pengetesan nya kita akan hit domain tersebut dari local kita dan meng check apakah traffic masuk ? apakah ada downtime ?

Untuk pengetesan ada di link ini ya

[https://youtu.be/lDYH4kFIj8g](https://youtu.be/lDYH4kFIj8g)

> _Panel kiri adalah cluster GCP ( Utama ) , Panel kanan adalah cluster Hetzner ( Backup )_

Dari video tersebut log 200 muncul di panel kiri yang artinya traffic sesuai mengarah ke cluster utama , untuk validasi berikut traffic ingress request

<p style="text-align: center"><img src="https://miro.medium.com/v2/resize:fit:700/1*s2NwB_VBIc_8X0L1aoy-xA.png"></p>

Sekarang menuju ke skenario , skenario nya adalah seolah — olah cluster pertama mengalami down atau trouble yang lain nya . Untuk membuat trouble kita cukup stop service haproxy di cluster utama dengan command

```
systemctl stop haproxy.service
```

> _Saat stop service haproxy pastikan local telah melakukan hit terus menerus agar ketauan apakah ada downtime atau tidak_

Berikut link demo nya

[https://youtu.be/KJYAd\_UC1ys](https://youtu.be/KJYAd_UC1ys)

Dari video tersebut kita lihat , Dari Cluster utama (panel kiri) menghasilkan log yang berarti traffic ke cluster utama , setelah membuat skenario seolah — olah cluster tersebut down yang mana meng stop service haproxy nya , Log dari cluster backup (panel kanan) mulai berjalan yang mana berarti traffic semua mengarah ke cluster backup dan saat haproxy cluster utama di jalankan , traffic normal seperti biasa ke cluster utama . Untuk validasi check dashboard ingress lagi

![](https://miro.medium.com/v2/resize:fit:700/1*_M-d8gaE5jvcP-7Yt6Y62Q.png)

Terjadi no traffic saat haproxy utama di stop

![](https://miro.medium.com/v2/resize:fit:700/1*1ov8ANx08zXR3ByFTKtl4A.png)

Terjadi traffic saat haproxy utama di stop .

Voilaaa , DR Traffic telah berhasil . Mungkin untuk setup tersebut hanya DR Traffic dengan topologi yang sangat simple , mungkin apabila sudah menuju ke prod atau dll , setup tersebut mungkin lebih komplex . Seharus nya DR dari mulai Deployment,DB semua itu sync antara cluster utama dan backup , karena untuk membuat itu cukup sangat panjang dan masih mencari bestpractice untuk sync DB nya mungkin saya bahas lain kali saja . Mohon maaf apabila ada kalimat yang susah dipaham / kurang mengerti / terlalu berbelit2 , sekian ges chersss. ☕