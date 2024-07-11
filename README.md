# Laporan Final Project Skalabilitas dan Reliabilitas Sistem

## Anggota Kelompok
| NRP | Nama | IP Server (VM) |
| ------ | ------ | ------ |
| 5027211017 | Arfan Yusran | 10.15.40.63 |
| 5027211049 | Tridiktya Hardani Putra | 10.15.40.62 |

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
    - [Instalasi Helm](#instalasi-helm)
    - [Menyiapkan Monitoring Environment](#menyiapkan-monitoring-environment)
      - [Tambahkan Helm Stable Charts untuk mesin lokal.](#tambahkan-helm-stable-charts-untuk-mesin-lokal)
      - [Menambahkan repositori Helm dari prometheus ke mesin lokal](#menambahkan-repositori-helm-dari-prometheus-ke-mesin-lokal)
      - [Membuat namespace tempat kita menginstall prometheus](#membuat-namespace-tempat-kita-menginstall-prometheus)
      - [Install kube-prometheus stack](#install-kube-prometheus-stack)
      - [Cek Pod](#cek-pod)
    - [Enable akses eksternal ke Infrastruktur](#enable-akses-eksternal-ke-infrastruktur)
    - [Buka Grafana untuk Memonitoring](#buka-grafana-untuk-memonitoring)
  - [Load Testing Sistem](#load-testing-sistem)
    - [Instalasi Locust](#instalasi-locust)
    - [Hasil Load Testing Locust](#hasil-load-testing-locust)
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
### Instalasi Helm

```
curl -fsSL -o get_helm.sh \ https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 

chmod 700 get_helm.sh 

./get_helm.sh
```
### Menyiapkan Monitoring Environment
#### Tambahkan Helm Stable Charts untuk mesin lokal.
```
helm repo add stable https://charts.helm.sh/stable
```
#### Menambahkan repositori Helm dari prometheus ke mesin lokal
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
#### Membuat namespace tempat kita menginstall prometheus
```
kubectl create namespace prometheus
```
#### Install kube-prometheus stack
```
helm install stable prometheus-community/kube-prometheus-stack --version 48.3.1 -n prometheus
```
#### Cek Pod
```
kubectl get pods -n prometheus
kubectl get svc -n prometheus
```
![image](https://github.com/trdkhardani/laporan-fp-skalabilitas/assets/99706251/f8a742cb-96ab-4028-8b7b-94060c6e8957)
![image](https://github.com/trdkhardani/laporan-fp-skalabilitas/assets/99706251/2c609d6b-b274-466a-93c3-60793c2450b1)

### Enable akses eksternal ke Infrastruktur
untuk melakukan hal tersebut, kita harus mengedit service Prometheus dan Grafana
```
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
kubectl edit svc stable-grafana -n prometheus
```
lalu ubah Type menjadi 'LoadBalancer' pada kedua service
![image](https://github.com/trdkhardani/laporan-fp-skalabilitas/assets/99706251/64baff10-27f6-48fb-a0de-8422ce97a00b)

### Buka Grafana untuk Memonitoring
Dikarenakan alamat ip eksternal grafana tidak muncul, kami melakukan forward proxy dari CLUSTER-IP grafana
![image](https://github.com/trdkhardani/laporan-fp-skalabilitas/assets/99706251/cbfbf576-a12b-49e0-bab9-ff8b613d5801)
![image](https://github.com/trdkhardani/laporan-fp-skalabilitas/assets/99706251/103707e9-c1ac-4329-8270-7e8b07e66d3d)

pada dashboard grafana tidak muncul hasil analisis dikarenakan kami mendapatkan kendala dimana webnya tiba-tiba error.
![image](https://github.com/trdkhardani/laporan-fp-skalabilitas/assets/99706251/d75d7c22-aae2-432a-908c-a87eaf0d60e2)


## Load Testing Sistem
### Instalasi Locust
Pada bagian testing ini kami menggunakan locust, dimana kita diharuskan menginstall ```pip install locust``` terlebih dahulu.

Lalu bisa membuat file locust.py
```
from locust import HttpUser, task, between

class HelloWorldUser(HttpUser):
    wait_time = between(0.5, 2.5)
    # a = 0
    # b = 0
    # c = 0 

    @task
    def test_index(self):
        response = self.client.get('/')
        message = response.json()['message']
        # print(message)
        # if message == 'This is server A':
        #     self.a += 1
        # elif message == 'This is server B':
        #     self.b += 1
        # elif message == 'This is server C':
        #     self.c += 1
        
        # print(f'A: {self.a}, B: {self.b}, C: {self.c}')
    
    @task
    def test_fast(self):
        self.client.get('/fast')
    
    @task
    def test_slow(self):
        self.client.get('/slow')
    
    @task
    def test_slow(self):
        self.client.get('/all')

    @task
    def test_get_id(self):
        response = self.client.get('/get/[ID]')
        message = response.json()['message']
        # if message == 'A':
        #     self.a += 1
        # elif message == 'B':
        #     self.b += 1
        # elif message == 'C':
        #     self.c += 1
        
        # print(f'A: {self.a}, B: {self.b}, C: {self.c}')
```
### Hasil Load Testing Locust
Hasilnya disini lagi-lagi tidak ada karena websitenya error
![image](https://github.com/trdkhardani/laporan-fp-skalabilitas/assets/99706251/8e7f6244-bd56-4ed8-b35f-250e697685f5)


## Analisis dan Kesimpulan
Pada laporan kali ini kami tidak bisa mendapatkan hasil analisis dan kesimpulan dikarenakan mendapat kendala yang akan dijelaskan secara detail pada seksi "Kendala" berikut.


## Kendala
Awalnya, aplikasi berjalan sebagaimana seharusnya dengan menggunakan port 8010 untuk reverse proxy ke **sa-frontend**, dan port 8080 untuk reverse proxy ke sa-webapp. Pada waktu itu, pods untuk **sa-web-app** dan **sa-logic**, masing-masing yang running atau berjalan hanya satu dari dua. Namun, ketika kami cek pods untuk melakukan konfigurasi monitoring yang dimana ternyata semua pods sudah running atau berjalan, tiba-tiba sa-frontend tidak dapat melakukan fetch ke sa-webapp karena masalh CORS. Berikut adalah error yang muncul:

![error-cors](/images/error-cors.png)

Beberapa pertemuan sebelumnya dalam perkuliahan, kami menemui masalah serupa, yaitu menggunakan port selain 80 untuk reverse proxy dan terjadi error CORS. Kami berhasil menyelesaikan masalah tersebut dengan mengganti ke port 80 berdasarkan saran dari Pak Fuad selaku dosen pengampu mata kuliah ini. Saat ini, masalahnya terletak pada port 80 yang secara 'misterius' digunakan oleh service tertentu, benar-benar tidak bisa dilakukan override dengan Nginx bahkan Docker. Berikut adalah tampilan error saat kami mencoba mengakses URL VM kami dengan port 80:

![error-port-80](/images/error-port-80.png)

Kami sudah memeriksa service yang menggunakan port 80 dengan perintah `sudo lsof -i :80` dan tidak menemukan service yang menggunakan port 80. Kami juga memeriksa dengan perintah `netstat -pln` dan tidak menemukan service apapun yang menggunakan port 80. Berikut adalah hasil dari perintah tersebut:

**VM/Node Master (10.15.40.62):**

![port-80-check-master-node](/images/port-80-check-master-node.png)

**VM/Node Worker (10.15.40.63):**

![port-80-check-worker-node](/images/port-80-check-worker-node.png)
