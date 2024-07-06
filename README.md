# Laporan Final Project Skalabilitas dan Reliabilitas Sistem

## Anggota Kelompok
| NRP | Nama |
| ------ | ------ |
| 5027211017 | Arfan Yusran |
| 5027211049 | Tridiktya Hardani Putra |

## Daftar Isi
- [Laporan Final Project Skalabilitas dan Reliabilitas Sistem](#laporan-final-project-skalabilitas-dan-reliabilitas-sistem)
  - [Anggota Kelompok](#anggota-kelompok)
  - [Daftar Isi](#daftar-isi)
  - [Pendahuluan](#pendahuluan)
  - [Arsitektur dan Implementasi Sistem](#arsitektur-dan-implementasi-sistem)
    - [Arsitektur Sistem](#arsitektur-sistem)
    - [Implementasi Sistem](#implementasi-sistem)
      - [Instalasi k3s di VM Master (10.15.40.62)](#instalasi-k3s-di-vm-master-10154062)
      - [Periksa apakah k3s di VM Master sudah berjalan](#periksa-apakah-k3s-di-vm-master-sudah-berjalan)
      - [Cek node-token di VM Master:](#cek-node-token-di-vm-master)
      - [Instalasi k3s di VM Worker (k3s-agent) (10.15.40.63)](#instalasi-k3s-di-vm-worker-k3s-agent-10154063)
      - [Jalankan file .yaml untuk deployment dan service sa-frontend di VM Master](#jalankan-file-yaml-untuk-deployment-dan-service-sa-frontend-di-vm-master)
      - [Dapatkan informasi URL sa-frontend](#dapatkan-informasi-url-sa-frontend)
      - [Lakukan Reverse Proxy ke URL sa-frontend pada k3s (Kubernetes) di VM Master](#lakukan-reverse-proxy-ke-url-sa-frontend-pada-k3s-kubernetes-di-vm-master)
      - [Di VM Worker, copy k3s.yaml dari VM Master](#di-vm-worker-copy-k3syaml-dari-vm-master)
      - [Build dan push sa-webapp di VM Worker](#build-dan-push-sa-webapp-di-vm-worker)
      - [Jalankan file .yaml untuk deployment dan service sa-webapp di VM Worker](#jalankan-file-yaml-untuk-deployment-dan-service-sa-webapp-di-vm-worker)
      - [Dapatkan informasi URL sa-webapp](#dapatkan-informasi-url-sa-webapp)
      - [Lakukan Reverse Proxy ke URL sa-webapp pada k3s (Kubernetes) di VM Worker](#lakukan-reverse-proxy-ke-url-sa-webapp-pada-k3s-kubernetes-di-vm-worker)
      - [Build dan push sa-logic di VM Worker](#build-dan-push-sa-logic-di-vm-worker)
      - [Jalankan file .yaml untuk deployment dan service sa-logic di VM Worker](#jalankan-file-yaml-untuk-deployment-dan-service-sa-logic-di-vm-worker)
  - [Monitoring Sistem](#monitoring-sistem)
  - [Load Testing Sistem](#load-testing-sistem)
  - [Analisis dan Kesimpulan](#analisis-dan-kesimpulan)
  - [Kendala](#kendala)

## Pendahuluan
Pada FP mata kuliah Skalabilitas dan Reliabilitas Sistem, kami diminta membuat sistem [sentiment analysis](https://github.com/rinormaloku/k8s-mastery) yang lebih scalable menggunakan banyak nodes. Kami menggunakan **k3s** sebagai kubernetes distribution yang lebih ringan dan cepat.

## Arsitektur dan Implementasi Sistem
### Arsitektur Sistem
Berikut adalah arsitektur sistem yang kami gunakan:

![arsitektur-sistem](/images/arsitektur-sistem.png)

**Master Node (10.15.40.62)** berisi frontend, sedangkan **Worker Node (10.15.40.63)** berisi backend/webapp dan Logic.

### Implementasi Sistem
#### Instalasi k3s di VM Master (10.15.40.62)
`curl -sfL https://get.k3s.io | sh -`

#### Periksa apakah k3s di VM Master sudah berjalan
`systemctl status k3s`

#### Cek node-token di VM Master:
`sudo cat /var/lib/rancher/k3s/server/node-token`

#### Instalasi k3s di VM Worker (k3s-agent) (10.15.40.63)
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.15.40.62:6443 K3S_TOKEN=K10034b3cc6970dd7975da4caea350fc0bd5938887e911fd738d77449a4b30562a9::server:0a301bb1a56da8177ee6c08afd364c24 K3S_NODE_NAME=skalabilitas-worker sh -```

#### Periksa apakah VM Worker telah menjadi agent atau worker untuk VM Master
```bash
k3s kubectl get nodes```

#### Build dan push sa-frontend di VM Master
```bash
yarn build
docker build -f Dockerfile -t tridiktyahp/sentiment-analysis-frontend .
docker push tridiktyahp/sentiment-analysis-frontend
```

#### Jalankan file .yaml untuk deployment dan service sa-frontend di VM Master
```bash
k3s kubectl apply -f sa-frontend-deployment.yaml
k3s kubectl apply -f service-sa-frontend-lb.yaml
```

#### Dapatkan informasi URL sa-frontend
`k3s kubectl get svc`

#### Lakukan Reverse Proxy ke URL sa-frontend pada k3s (Kubernetes) di VM Master
```bash 
server {
        listen 8010;
        listen [::]:8010;

        index index.php index.html index.htm index.nginx-debian.html;

        try_files $uri $uri/ /index.php$is_args$args;

        server_name _; 
location / {
                proxy_pass http://10.43.101.161;
        }


        # pass PHP scripts to FastCGI server
        #
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
        #       proxy_pass http://worker;
        #
        #       # With php-fpm (or other unix sockets):
                fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        #       # With php-cgi (or other tcp sockets):
        #       fastcgi_pass 127.0.0.1:9000;
        }

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        location ~ /\.ht {
                deny all;
        }
        error_log /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log;
}
```

#### Di VM Worker, copy k3s.yaml dari VM Master
Sebelum melakukan copy, ubah izin file k3s.yaml di VM Master: ```sudo chmod 644 /etc/rancher/k3s/k3s.yaml.```

```bash
scp skalabilitas@10.15.40.62:/etc/rancher/k3s/k3s.yaml /home/skalabilitas/.kube/config
```

Ubah isi `server` pada `/home/skalabilitas/.kube/config` di VM Worker dengan `https://10.15.40.62:6443`

Setelah itu, ubah izin file **k3s.yaml** seperti semula di **VM Master**: `sudo chmod 600 /etc/rancher/k3s/k3s.yaml`

#### Build dan push sa-webapp di VM Worker
```bash
docker build -f Dockerfile -t jezz16/sentiment-analysis-web-app .
docker push jezz16/sentiment-analysis-web-app
```

#### Jalankan file .yaml untuk deployment dan service sa-webapp di VM Worker
```bash
k3s kubectl apply -f sa-web-app-deployment.yaml
k3s kubectl apply -f service-sa-web-app-lb.yaml
```

#### Dapatkan informasi URL sa-webapp
`k3s kubectl get svc`

#### Lakukan Reverse Proxy ke URL sa-webapp pada k3s (Kubernetes) di VM Worker
```bash 
server {
       listen 8080;
       listen [::]:8080;

       server_name _;
#
#       root /var/www/example.com;
#       index index.html;

       location / {

#                add_header 'Access-Control-Allow-Origin' '*';
#                add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept';
#                add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
                proxy_pass http://10.43.193.149/;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
        }
}
```

#### Build dan push sa-logic di VM Worker
```bash
docker build -f Dockerfile -t jezz16/sentiment-analysis-logic .
docker push jezz16/sentiment-analysis-logic
```

#### Jalankan file .yaml untuk deployment dan service sa-logic di VM Worker
```bash
k3s kubectl apply -f sa-logic-deployment.yaml
k3s kubectl apply -f service-sa-logic.yaml
```

## Monitoring Sistem

## Load Testing Sistem

## Analisis dan Kesimpulan

## Kendala
Awalnya, aplikasi berjalan sebagaimana seharusnya dengan menggunakan port 8010 untuk reverse proxy ke **sa-frontend**, dan port 8080 untuk reverse proxy ke sa-webapp. Pada waktu itu, pods untuk **sa-web-app** dan **sa-logic**, masing-masing yang running atau berjalan hanya satu dari dua. Namun, ketika kami cek pods untuk melakukan konfigurasi monitoring yang dimana ternyata semua pods sudah running atau berjalan, tiba-tiba sa-frontend tidak dapat melakukan fetch ke sa-webapp karena masalh CORS. Berikut adalah error yang muncul:

![error-cors](/images/error-cors.png)

Beberapa pertemuan sebelumnya dalam perkuliahan, kami menemui masalah serupa, yaitu menggunakan port selain 80 untuk reverse proxy dan terjadi error CORS. Kami berhasil menyelesaikan masalah tersebut dengan mengganti ke port 80 berdasarkan saran dari Pak Fuad selaku dosen pengampu mata kuliah ini. Saat ini, masalahnya terletak pada port 80 yang secara 'misterius' digunakan oleh service tertentu, benar-benar tidak bisa dilakukan override dengan Nginx bahkan Docker. Berikut adalah tampilan error saat kami mencoba mengakses URL VM kami dengan port 80:

![error-port-80](/images/error-port-80.png)