# Catatan
Gw rapihin poin yang bisa dipetik dan "dipelajari" karena inti dari cloud_pricing adalah ngitung atau reka estimasi biaya cloud yang akan kita bayar kira-kira berapa

# Skenario
Contohnya nih
aplikasi butuh 1 pod = 1vCPU + 2GB Ram (bisa t2.small)
dan butuh replika 4
berarti total resource 4 vCPU + 8GB Ram (bisa 2x t2.medium atau t2.large)

> enggak tau dah kenapa butuh 1vCPU + 2GB Ram. Yang jelas pasti ada requirement minimal banget mereka butuh "segitu"

pertanyaannya, 
1. kenapa replica sedangkan bisa pakai 1 pod = 4 vCPU + 8GB Ram

jawabannya,
Bisa aja, secara teknis valid dan sering disebut sebagai scale-up (besar tapi tunggal), tapi ada pertanyaan lagi
Kenapa biasanya tetap direkomendasikan multiple replicas?
Jawabannya:
1. high reliability, misal satu pod lagi down (entah crash dan lainnya), sisanya ada 3 yang masih bisa dipakai sehingga aplikasi tetap berjalan.
2. load balancing, banyak user = lebih efisien dibagi ke banyak pod daripada overload satu pod. Berani satu pod loadnya besar? ga apa-apa sih.
3. rolling update, k8s bisa rolling dengan zero downtime kalau ada beberapa pod. 
4. node utilization, kadang node kecil lebih efisien secara cost/penjadwalan, dan bisa autoscale.
5. latency, kalau pod tersebar secara geografis (multi-zone), latency ke user bisa turun.

Menurut gw pribadi, dari 4 sampai 5 kalau sudah beda region cloud harganya juga variatif. Kalau 3 juga oke sih, tapi gw pikir-pikir bakalan ada cost untuk k8s sendiri. Lanjut ya.

> Ada alasan yang lebih-lebih lagi loh! Nanti gw angkat dibawah ya

# Pricing
> tl;dr selain pertimbangan kenapa butuh lebih dari satu instance. Tolong hitung kira-kira berapa juga "biaya" yang enggak terduga selain hitung-hitungan tarif, variabel sebannya lumayan.

Ngomongin pricing. kenapa enggak 1 pod aja? Karena
1. lebih murah,
2. tidak tahan gangguan

Nah ini, dari poin kedua yang mengganjal. Makanya, umumnya satu pod aja cukup kalau
1. dev/test environment
2. aplikasi internal
3. tidak butuh HA (high availability)
4. sudah punya layer HA yang lain (misal di reverse proxy/LB)

Oke kalau pod replika?
1. ~~agak maehong (mahal), tapii: fault tolerance, load balancing, dan skalabilitas~~ dengan jenis instance yang sama jatuhnya lebih mahal sih iya, secara reliabilitas bisa dipertimbangkan, kalau secara billing ini bisa bikin nangis kalau enggak dijaga

Itung-itungan yuk. Lanjut dari atas tadi, dan gw pakai AWS

| Instance  | vCPU | RAM  | Harga (\~USD/hour) |
| --------- | ---- | ---- | ------------------ |
| t2.micro  | 1    | 1 GB | \~\$0.0116         |
| t2.small  | 1    | 2 GB | \~\$0.023          |
| t2.medium | 2    | 4 GB | \~\$0.0464         |
| t2.large  | 2    | 8 GB | \~\$0.0928         |

Untuk 4 pod yang masing-masing butuh 1 vCPU + 2GB RAM, kita bisa kira-kira pakai 2 x t2.medium

Tapi, hitungan diatas hanya sekadar instancenya aja, belum termasuk komponen yang lain, yaitu:
1. EKS Control Plane
+ ~$0.10/hour = $2.40/day (ini flat fee per EKS cluster)

2. Load Balancer
+ ~ $0.025/hour = $0.60/day

3. EBS (disk)
+ Default ~8GB per node
+ ~ $0.10/GB-month → 16GB = $0.05/day

## Kenapa butuh ketiga item tadi?
1. EKS: Kalau pakai Kubernetes, eks adalah soiusi managed-nya. Enggak perlu ngatur control plane/manual cluster k8s 
2. EBS: penyimpanan yang persistence di node. Kalau butuh volume mount untuk pod (misalnya PostgreSQL, redis dengan persistence), nah ini butuh EBS
3. Load Balancer: ekspose aplikasi ke Internet, atau pakai Ingress Controller kayak NGINX/ALB yang otomatis bikin Load Balancer

Tapii...
1. EKS ada biaya tetap $0.10/jam ($2.40/hari)
2. Kalau pakai EC2 biasa + kubeadm emang bisa hemat, tapi mesti setup sendiri semuanya (lebih ribet, raw)
3. Kalau aplikasinya stateless, enggak perlu bikin volume mount tambahan.

Btw, bukannya EC2 ada volume sendiri dan itu default? Iya sih, karena AWS otomatis attach EBS sebagai root disk (tempat OS dan data disimpan). Dengan itu, kita bisa pilih ukuran, dan tetap dihitung biaya storage per GB per bulan.

> katanya T2, T3, T4g enggak punya instance store, jadi wajib pakai EBS. Dan kalau enggak tau EKS, saran gw tahan dan kecualikan terlebih dulu

# Simulasi biaya per hari
| Komponen            | Jumlah          | Biaya/Hari (USD)  |
| ------------------- | --------------- | ----------------- |
| EC2 t2.medium       | 2 node          | \~\$2.23          |
| EKS Control Plane   | 1 cluster       | \~\$2.40          |
| Load Balancer (ALB) | 1 unit          | \~\$0.60          |
| EBS for app storage | 3 x 10GB = 30GB | \~\$0.08          |
| **Total Estimasi**  |                 | **\~\$5.31/hari** |

atau 

| Komponen            | Biaya/bulan (USD)  | Biaya/tahun (USD)   |
| ------------------- | ------------------ | ------------------- |
| EC2 (2 t2.medium)   | \$2.23 × 30 = 66.9 | \$66.9 × 12 = 802.8 |
| EKS Control Plane   | \$2.40 × 30 = 72   | \$72 × 12 = 864     |
| Load Balancer (ALB) | \$0.60 × 30 = 18   | \$18 × 12 = 216     |
| EBS Storage (30GB)  | \$0.08 × 30 = 2.4  | \$2.4 × 12 = 28.8   |
| **Total**           | **\$159.30**       | **\$1,911.60**      |

## Ada yang lebih hemat lagi?
Ada dong, kalau elu yang mikirin perusahaan atau dompet sendiri bisa pertimbangkan ini sih kalau kata gw:
| Komponen          | Opsi Hemat                               | Catatan                                                                     |
| ----------------- | ---------------------------------------- | --------------------------------------------------------------------------- |
| **EC2 Node**      | Pakai instance lebih kecil (t2.micro)    | Bisa hemat 3-4x, tapi resource sangat terbatas, kurang ideal produksi.      |
| **EKS**           | Setup Kubernetes sendiri (self-managed)  | Bisa hemat biaya control plane \$2.4/hari, tapi effort sangat tinggi.       |
| **Load Balancer** | Pakai NodePort + NAT Gateway / VPN       | Bisa hilangkan biaya ALB, tapi akses public lebih ribet & kurang fleksibel. |
| **EBS Storage**   | Gunakan storage external seperti S3, EFS | Bisa hemat biaya EBS, tapi aplikasi harus support remote storage.           |

# Kesimpulan Sementara
Menurut gw sih itu estimasi yang minimal cukup realistis untuk setup Kubernetes di AWS dengan:
+ 2 node EC2 t2.medium (cukup kecil dan murah untuk node produksi yang bisa jalan 3 pod),
+ 1 cluster EKS managed (biaya control plane fix),
+ 1 Application Load Balancer (ALB) untuk expose service,
+ EBS storage 10GB per pod (cukup untuk app yang butuh persistensi).

# Case lain nih
Misal, gw ga pengen dulu pakai Kubernetes karena belum mateng dalam mengambil opsi tersebut, bisa lebih murah dong pastinya. Dan ini komponen tanpa Kubernetes
| Komponen          | Penjelasan                         | Biaya Perkiraan                                                                      |
| ----------------- | ---------------------------------- | ------------------------------------------------------------------------------------ |
| **EC2 Instance**  | Server virtual untuk aplikasi      | Bayar per jam sesuai instance type (misal t2.medium ≈ \$0.0464/jam → \~\$33.4/bulan) |
| **EBS Volume**    | Storage untuk OS + data            | Root volume biasanya 8-30GB, sekitar \$0.08/GB/bulan                                 |
| **Elastic IP**    | IP publik (jika dipakai)           | Gratis kalau dipakai terus, bayar kalau idle                                         |
| **Load Balancer** | Opsional, untuk traffic distribusi | Kalau perlu, bayar sesuai pemakaian (\~\$18/bulan untuk ALB dasar)                   |

## Totalnya?
| Komponen            | Jumlah          | Harga per unit          | Total per bulan                 | Total per tahun            |
| ------------------- | --------------- | ----------------------- | ------------------------------- | -------------------------- |
| EC2 t2.medium       | 3 instances     | \$0.0464 per jam        | 0.0464 × 24 × 30 × 3 = \$100.22 | \$100.22 × 12 = \$1,202.64 |
| EBS gp3 (30GB)      | 3 × 30GB = 90GB | \$0.08 per GB per bulan | 90 × 0.08 = \$7.20              | \$7.20 × 12 = \$86.40      |
| ALB (Load Balancer) | 1               | \~\$18 per bulan        | \$18.00                         | \$216.00                   |

## Bedanya?
| Komponen          | Pakai EKS (per bulan) | Tanpa EKS (per bulan)           | Selisih (hemat)                                                                |
| ----------------- | --------------------- | ------------------------------- | ------------------------------------------------------------------------------ |
| EC2 Nodes         | \$2.23 × 2 × 30 = 134 | \$0.0464 × 24 × 30 × 3 = 100.22 | \~\$33.78 lebih mahal di EKS (karena node lebih sedikit tapi pod lebih banyak) |
| EKS Control Plane | \$2.40 × 30 = 72      | \$0                             | **\$72 hemat tanpa EKS**                                                       |
| Load Balancer     | \$0.60 × 30 = 18      | \$18                            | Sama                                                                           |
| EBS Storage       | \$0.08 × 30 = 2.4     | \$0.08 × 90 = 7.2               | \$4.8 lebih hemat di EKS (karena storage lebih sedikit: 3x10GB vs 3x30GB)      |
| **Total**         | **\$159.30**          | **\$125.42**                    | **\~\$33.88 hemat tanpa EKS**                                                  |

## Ada resikonya yang perlu dipelajari dulu nih
Apa Resikonya Tanpa Kubernetes?
+ Tidak ada orchestration otomatis
+ Tidak ada autoscaling built-in
+ Kalau instance mati, aplikasi juga mati
+ Deployment dan scaling harus manual

Eh loh, loh. Kok "Kalau instance mati, aplikasi juga mati?" Nah itulah kita manfaatkan Load Balancer (ALB/NLB), cuman kurangnya orkestrasi aja

> Ini semua prakiraan pricing per bulan diluar dari hal-hal yang tidak terduga. Salah satunya entah komputasi, entah traffic, entah bandwitdh

# Biaya diluar yang tak terduga
Satu-satu deh perkiraan yang bisa bikin "mehong" diluar dari prakiraan tak-terduga
1. Data Transfer
+ Data keluar (egress) dari AWS ke internet atau antar region dikenakan biaya
+ Data transfer antar AZ (Availability Zone) juga dikenakan biaya (meski lebih murah)
+ Traffic yang tinggi ke Load Balancer (ALB) juga kena biaya berdasarkan GB yang diproses

2. Autoscaling / Spikes Usage
+ Hati hati pakai autoscalling, tiba-tiba traffic naik jadi AWS akan otomatis spin-up lebih banyak instance yang buat biaya jadi naik
+ Beban komputasi yang lebih berat juga bikin resource lebih cepat habis, dan autoscaling aktif

3. Request & API Calls
+ Beberapa layanan seperti EKS juga mengenakan biaya untuk API call ke control plane
+ Load Balancer juga bisa dikenakan biaya berdasarkan jumlah request

4. Snapshot & Backup
+ Kalau rutin ambil snapshot EBS atau backup ke S3, itu juga ada biaya storage dan request

5. Resource yang Terbuang
+ Misal, kamu punya instance tapi idle (tidak dipakai maksimal), tetap bayar penuh
+ Storage yang tidak terpakai tapi masih ada di volume tetap kena biaya

## Casenya apa nih?
Misal traffic aplikasinya tetiba-tiba naik dari 100GB data keluar jadi 500GB, dengan biaya data keluar sekitar $0.09/GB, berarti:
100GB × $0.09 = $9
500GB × $0.09 = $45
Naik $36 di billing cuma dari data transfer!

## Biar enggak tetiba naik gimana nih?
Tipsnya ya begini sih kira-kira:
+ Monitor penggunaan traffic dan resource pakai CloudWatch
+ Autoscaling dengan threshold yang tepat agar tidak overprovision
+ Menggunakan caching dan CDN (CloudFront) untuk kurangi data keluar langsung dari server
+ Rutin cleanup resource yang tidak dipakai

### Ada opsi lain selain CloudWatch
Yaitu Grafana + Prometheus. Eh inget, dari case kita kan butuh 4 replika dan itu berarti dua instance t2.medium. Gimana dong?
Sarannya sih kayak gini kalau mau pasang:
+ Aplikasi (4 replika) hanya perlu expose metrics endpoint (misalnya /metrics ala Prometheus format)
+ Prometheus (1 instance saja) akan melakukan scrape (pull) ke semua endpoint pod aplikasi tersebut
+ Grafana cukup satu juga, cukup pointing ke Prometheus sebagai data source

dan di config prometheusnya kira-kira begini
```yaml
scrape_configs:
  - job_name: 'my-app'
    static_configs:
      - targets: ['10.0.0.11:8080', '10.0.0.12:8080', '10.0.0.13:8080', '10.0.0.14:8080']
```

Oh iya, ngomongin soal komponennya. Begini kira-kira
| Komponen          | Fungsi                                                             |
| ----------------- | ------------------------------------------------------------------ |
| **Prometheus**    | Tarik & simpan metrik (EC2, aplikasi, dll)                         |
| **Grafana**       | Tampilkan dashboard, bikin alert, dsb                              |
| **Node Exporter** | Ekstensi ringan buat expose metrics dari EC2 (CPU, RAM, disk, dsb) |

+ Prometheus bisa scrape dari semua EC2 (via node_exporter)
+ Grafana ambil data dari Prometheus
+ Semuanya bisa jalan di 1 instance kecil (misal t2.small) buat hemat
+ Bisa backup Prometheus TSDB ke S3 (opsional)

Estimasi biaya?
| Komponen           | Jumlah     | Estimasi Biaya (USD)              |
| ------------------ | ---------- | --------------------------------- |
| EC2 t2.small       | 1 instance | \~\$0.023 per jam = \~\$16.50/bln |
| Storage (EBS 20GB) | 1 volume   | \~\$1.60/bulan                    |
| **Total**          |            | **\~\$18.10 per bulan**           |

> Diatas tadi sebenarnya bisa "ngikut" di satu instance, untuk menekan harga billing ~$18.10 per bulan itu, hitung-hitung dapat forecast kemampuan aplikasi kita atau antisipasi dini bahwa trafficnya bakal gede/besar.

# Kesimpulan, bener deh
Menurut gw, selain pricing yang ada dan perkiraan sebab billing yang naik udah cukup sih.