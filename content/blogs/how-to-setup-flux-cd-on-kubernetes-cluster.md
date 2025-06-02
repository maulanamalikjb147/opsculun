---
author: Maulana malik jabbar budianto
title: How to setup Flux CD On Kubernetes Cluster
description: How to setup Flux CD On Kubernetes Cluster
date: 2025-06-02T20:15:00+07:00
cover:
  image: /images/0_jBtzeAx-2mF02NR_.webp
showToc: true
tocopen: false
tags:
  - kubernetes
---
Flux CD adalah alat yang digunakan untuk otomatisasi pengelolaan konfigurasi dan pengiriman aplikasi di lingkungan Kubernetes. Flux CD memungkinkan Continuous Delivery (CD) pada cluster Kubernetes dengan memonitor dan mengelola perubahan pada konfigurasi Kubernetes Anda.

Mungkin secara singkat tentang flux adalah begitu 😁 , untuk selengkap nya bisa berkunjung saja ke website resmi nya [https://fluxcd.io/](https://fluxcd.io/)

Selain itu flux juga ada banyak beberapa fitur yaitu :  
Sinkronisasi Otomatis: Flux secara terus-menerus memonitor repositori Git yang berisi konfigurasi Kubernetes Anda. Ketika ada perubahan di repositori, Flux akan mendeteksi perubahan tersebut dan menerapkannya ke cluster Kubernetes.

1.  Automated Rollbacks: Jika perubahan yang diterapkan menyebabkan masalah atau kesalahan, Flux dapat melakukan rollback otomatis ke versi sebelumnya.
    
2.  Automated Image Updates: Flux mendukung otomatisasi pembaruan gambar (image) container. Dengan konfigurasi yang benar, Flux dapat memantau perubahan gambar dan secara otomatis memperbarui aplikasi di cluster Anda.
    
3.  Deklaratif Konfigurasi: Flux menggunakan pendekatan deklaratif di mana konfigurasi aplikasi dinyatakan dalam berkas manifest Kubernetes (misalnya, file YAML). Perubahan pada berkas manifest ini akan direfleksikan pada cluster Kubernetes.
    
4.  Integrasi dengan GitOps: Flux sangat terkait dengan konsep GitOps, yang menekankan penggunaan repositori Git sebagai sumber kebenaran tunggal untuk konfigurasi dan definisi infrastruktur. Dengan GitOps, perubahan pada cluster Kubernetes direkam dalam repositori Git, memungkinkan rekam jejak perubahan dan konsistensi yang terjamin.
    

Selebih nya untuk fitur nya bisa cek website nya juga ya .

Disini saya akan share setup Flux dengan cluster Kubernetes nya agar proses CD berjalan lancar , tetapi disini saya setup dengan Sangat minimalist dan sangat sederhana 😁.

# **Setup Flux CD**

Install flux CD CLI

```
curl -s https://fluxcd.io/install.sh | sudo bash
```

Setup bash completion

```
. <(flux completion bash)
```

Install deployment flux , Install deployment ada berbanyak cara tapi saya menggunakan Flux CLI untuk deploy deployment nya

Menggunakan cli

```
flux install
```

Menggunakan kubectl

```
kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml
```

Menggunakan Helm

```
helm install -n flux-system flux oci://ghcr.io/fluxcd-community/charts/flux2
```

## **Integration Flux with cluster**

Boostrap flux with repository with gitlab , export token gitlab

```
export GITLAB_TOKEN=<gh-token>
```

```
flux bootstrap gitlab \
  --deploy-token-auth \
  --owner=my-gitlab-username \
  --repository=my-project \
  --branch=master \
  --path=clusters/my-cluster \
  --personal
```

> _Sesuaikan Info ini dengan repository kalian , apabila repository nya public bisa ganti — personal dengan — private=false._

Setelah berhasil boostrap si flux akan nge push directory dan manifest seperti ini

<p style="text-align: center"><img src="https://miro.medium.com/v2/resize:fit:700/1*LoinGMlZWNSmTLqvLdkMJw.png"></p>

pastikan cek kustomization flux-system terinstall

```
kubectl get kustomizations.kustomize.toolkit.fluxcd.io -n flux-system
```

<p style="text-align: center"><img src="https://miro.medium.com/v2/resize:fit:599/1*fQSzWFMHSabnO2h-j-TS-A.png"></p>

## **Setup Kustomization**

Buat secret untuk flux akses ke repository , karena gitlab menggunakan token jadi sample nya seperti ini

convert token to base64

```
echo -n "token-anda" | base64
```

```
nano secret-gitrepository.yaml
```

```
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-secret
  namespace: flux-system
type: Opaque
data:
  token: Z2xwYXQtRzhxTDJ4TXlIVsdawd212syenpweG1wS2k=
```

```
kubectl create -f secret-gitrepository.yaml
```

atau bisa juga dengan command seperti ini

```
flux create secret git gitlab-secret \
    --url=https://gitlab.com/blablabla/flux.git \
    --username=maulanamalik \
    --password=blablabla
```

> _Sesuaikan token dengan token gitlab kalian , usahakan hash dulu_

Buat manifest kustomization applications-1 dan git repository nya di **flux/clusters/my-cluster**

kustomization :

```
nano applications-1-kustomization.yaml
```

```
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: applications-1
  namespace: flux-system
spec:
  interval: 20s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./clusters/my-cluster/repo-apps/applications-1
  prune: true
  timeout: 1m
```

gitrepository:

```
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: applications-1
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: gitlab-secret
  timeout: 1m0s
  url: https://gitlab.com/opsculun/flux.git
```

> _Untuk path kustomization sesuaikan path nya dengan path deployment kalian di simpan di repository . dan secretref sesuaikan dengan nama secret yang tadi di deploy_

## **Example struktur deployment untuk apps**

File di bawah ini di simpan di path sesuai yang tadi telah di setup yaitu di **./clusters/my-cluster/repo-apps/applications-1**

deployment.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-applications-1
  namespace: applications-1
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
        image: nginx:latest
        ports:
        - containerPort: 80
```

service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-applications-1
  namespace: applications-1
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

kustomization.yaml

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - service.yaml
  - deployment.yaml
```

> _untuk kustomization.yaml usahakan penamaan resources sama dengan nama manifest deployment yang mau di deploy_

check kustomization applications-1 apakah sudah ada atau belum

<p style="text-align: center"><img src="https://miro.medium.com/v2/resize:fit:700/1*MwdjUzq9SLC4u989a3qNpw.png"></p>

nanti proses reconciliation applications-1 berjalan dan tunggu deployment ter deploy dan running

<p style="text-align: center"><img src="https://miro.medium.com/v2/resize:fit:700/1*qI8mNPgOpMxckg271UaODA.png"></p>

Apabila ada commit di manifest , otomatis deployment yang telah terdeploy akan ikut terupdate juga

Voilaaa , Setup Flux CD on Cluster kubernetes berhasill . Dari Setup di atas sangat simple sekali mungkin bisa di kreasikan tergantung kalian mau gimana proses CD nya , Mungkin kedepan nya per deployment tuh di bedakan branch jadi semisal branch application-1 hanya ada repository deployment application-1 sehingga menjadi rapi tidak menumpuk di satu branch . Mungkin nanti saya akan share full nya dari proses CI sampai CD nya 😁.Mohon maaf apabila ada kalimat yang susah dipaham / kurang mengerti / terlalu berbelit2 , sekian ges chersss. ☕