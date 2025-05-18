# CPU Credit?
Di AWS, CPU Credit adalah kredit yang bisa digunakan untuk utilisasi pemakaian CPU berdasarkan batas yang udah AWS berikan per jenis instance saat instance butuh pemakaian CPU yang lebih. Lebihnya tergantung:

Misalnya
| Instance type | CPU credits earned per hour | Maximum earned credits that can be accrued* | vCPUs*** | Baseline utilization per vCPU |
| ------------- | --------------------------- | ------------------------------------------- | -------- | -------------------------- |
| t3.small | 24 | 576 | 2 | 20%** |
| t3.medium	| 24 | 576 | 2 | 20%** |

\* The number of credits that can be accrued is equivalent to the number of credits that can be earned in a 24-hour period.

** The percentage baseline utilization in the table is per vCPU. In CloudWatch, CPU utilization is shown per vCPU. For example, the CPU utilization for a t3.large instance operating at the baseline level is shown as 30% in CloudWatch CPU metrics.

*** Each vCPU is a thread of either an Intel Xeon core or an AMD EPYC core, except for T2 and T4g instances.

dan CPU Credit tersebut dipakai per menit. (dengan asumsi CPU Usage adalah 100% per 1 menit)

# Insight yang bisa gw catet
Untuk dapat Maximum credit, instance perlu "didiamkan" selama 24 jam. Hitungannya: Maximum earned credits that can be accrued / CPU credits earned per hour = Total hour needed. Walaupun begitu, billing tetap berjalan.

Baseline utilization per vCPU adalah batas bawah tiap core vCPU supaya dapat CPU Credits

## Contoh
Satu instance `t3.medium` gw pakai dengan rata-rata vCPU dibawah 20%. Selama seharian, billing yang masuk ke gw tetep $0.0528 x 24 jam = $1,2672 per hari. Ngomongin cpu credit, gw dapet Credit sebanyak 576 poin.

Dari 576 poin tersebut, kalau gw pakai vCPU dengan kapasitas 100%, kredit gw akan habis dalam waktu kurang lebih ~4,8Jam, hitungannya dari 576 / 60 Menit = 4,8 Jam

Misalnya, ada kala dimana CPU Credit gw habis. Apa yang terjadi? Kena denda dong, dalam artian bayar biaya throttle. Untuk jenis instance EC2 T2, T3, dan T4g

> Ada insight, berapa besarannya kurang tau variabelnya. Yang jelas bisa sampai 3x dari tarif per jam instance tersebut