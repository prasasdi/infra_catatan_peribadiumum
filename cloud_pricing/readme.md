# Langsung aja ya ke skenario
contoh skenario
aplikasi butuh 1 pod = 1vCPU + 2GB Ram (bisa t2.small)

replika 4
berarti total resource 4 vCPU + 8GB Ram (bisa 2x t2.medium atau t2.large)

pertanyaannya, 
1. kenapa replica sedangkan bisa pakai 1 pod = 4 vCPU + 8GB Ram

jawabannya,
Bisa aja, secara teknis valid dan sering disebut sebagai scale-up (besar tapi tunggal), tapi ada pertanyaan lagi
Kenapa biasanya tetap direkomendasikan multiple replicas?
Jawabannya:
1. high reliability, misal satu pod lagi down (entah crash dan lainnya), sisanya ada 3 yang masih bisa dipakai sehingga aplikasi tetap berjalan
2. load balancing, banyak user = lebih efisien dibagi ke banyak pod daripada overload satu pod. Berani satu pod loadnya besar? ga apa-apa sih
3. 
