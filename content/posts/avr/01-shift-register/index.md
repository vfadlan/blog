---
date: 2025-12-28T05:15:34+07:00
draft: false
params:
  author: Fadlan Abduh
title: Animating LEDs with Shift Register
series: ["AVR"]
series_order: 1
---
Di blog kali ini, kita akan membahas tentang bagaimana menganimasikan 8 LED dengan shift register. Sebelum lanjut, saya mengasumsikan pembaca sudah memahami beberapa hal dasar tentang register, kita akan membahas ini berulang kali kedepannya. Jika ingin tahu, silakan baca blog sebelumnya atau sumber lain.
{{< article link="/posts/avr/00-hello/" showSummary=true compactSummary=true >}}

Jadi, bagaimana caranya menganimasikan 8 LED dengan mikrokontroler? Cara paling sederhana adalah menggunakan 8 pin GPIO dari port yang sama. Contoh, jika kita menggunakan Port B, maka kita perlu mengubah register DDRB sekali dan memanipulasi PORTB untuk setiap frame-nya. Selesai.

Tapi terdapat masalah lain, jumlah pin GPIO pada MCU sangatlah terbatas. Beberapa MCU bahkan hanya memiliki 6 GPIO. Atau, meskipun jumlah pin yang dimiliki melimpah, kita tidak tahu pada update versi selanjutnya akan membutuhkan berapa banyak pin.

Pada kasus kita, ATMega328P sebenarnya memiliki pin yang cukup untuk menangani 8 LED dengan port yang sama. _But, let's do it anyway_.

## Shift Register
Kata "register" pada shift register secara konsep sama saja seperti register yang ada pada MCU. Register is register. Yang membedakan adalah kata "shift" (geser). Jadi, shift register adalah salah satu jenis register yang bisa menggeser data di dalamnya dari bit 0 ke bit 1, bit 1 ke bit 2, dan seterusnya ketika diberi sinyal clock.

Di dalam MCU juga terdapat internal shift register, tapi kali ini kita akan membahas tentang external shift register dalam bentuk Integrated Circuit (IC). Karena ini eksternal, satu-satunya cara untuk memanipulasi register adalah dengan mengirimkan sinyal elektrik HIGH dan LOW. Alih-alih melakukan assignment seperti `PORTB = 0x3F;` pada program MCU.

Terdapat dua jenis shift register: Serial In Parallel Out (SIPO), dan Parallel In Serial Out (PISO). Yang akan kita gunakan adalah SIPO karena kita ingin mengurangi penggunaan pin MCU.

_Catatan: Mulai sekarang sampai blog ini selesai, SR berarti Shift Register. Perlu diingat bahwa terdapat makna lain dari SR._

## 74HC595 IC
### Overview
![Shift Register Illustration](img/working-sr.gif "Ilustrasi shift register, sumber lastminuteengineers.com")
74HC595 adalah Integrated Circuit yang berguna untuk mengubah input serial menjadi output paralel (SIPO) dengan kapasitas maksimal 8-bit. Di dalamnya, terdapat dua macam register:
* shift register berguna sebagai "antrean" data masuk,
* storage register (disebut juga latch) berguna untuk menyimpan data dari shift register dan diteruskan ke pin output. Data shift register hanya akan disimpan jika pin RCLK (disebut juga latch) diberi perubahan sinyal dari LOW ke HIGH.

### Pin Descriptions
![SN74HC595 DIP Pinout](img/sn74hc595-pinout.png "Pinout SNx4HC595, diambil dari datasheet Texas Instruments (TI). Perlu dicatat bahwa garis di atas pin OE dan RCLK memiliki Acrive Low, pin tersebut akan aktif jika dihubungkan ke ground.")
|Pin Number|Pin Name|Description|
|:--|:--|:--|
|15,1-7|Qa-Qh|Parallel Output, diambil dari storage register.|
|8|GND|Ground Pin.|
|9|Qh'|Serial Output untuk merangkai 2 shift register atau lebih (kita abaikan di blog ini).|
|10|SRCLR|Shift Register Clear, aktif ketika terhubung ground.|
|11|SRCLK|Shift Register Clock, akan menggeser data di dalam shift register jika pin ini diberi sinyal dari LOW ke HIGH (rising edge).|
|12|RCLK|Register Clock, saat diberi rising edge data shift register akan disalin ke storage register, sehingga mengubah output.|
|13|OE|Output Enable, aktif ketika terhubung ke ground| 
|14|SER|Serial data, jalur masuk data serial.|
|16|VCC|Suplai tegangan dari -0.5V sampai dengan 7V.|

### Functional Characteristics
![SN74HC595 Function Table](img/sn74hc595-function.png "Tabel mode fungsi SNx4HC595, diambil dari datasheet TI.")

Untuk memahami tabel di atas, berikut beberapa penjelasan untuk masing-masing jenis input:
* X: Don't care, tidak peduli input yang diberikan,
* H: HIGH,
* L: Low,
* ↑: Rising edge, ketika sinyal berubah dari LOW ke HIGH.

Pada tabel fungsi, perubahan data dipicu oleh sinyal clock. Yang diperhatikan bukan sinyal HIGH atau LOW, tetapi transisi dari LOW ke HIGH (rising edge). 
ss
Berikut adalah penjelasan masing-masing mode:
* Fungsi 1 dan 2, untuk mengaktifkan pin output (Qa-Qh), pin OE harus disambungkan ke ground.  
* Fungsi 3, kita perlu menghubungkan pin SRCLR ke VCC agar SR tidak di-clear.  
* Fungsi 4, ketika SER bernilai LOW, SRCLR HIGH, dan memberi SRCLK sinyal clock: SR akan bergeser dan bit pertama akan bernilai 0.
* Fungsi 5, Sama seperti sebelumnya, namun SER bernilai HIGH sehingga SR bergeser dan bit pertama bernilai 1.  
* Fungsi 6, ketika RCLK diberi sinyal clock (rising edge), data di dalam SR akan disalin ke storage register.

Jadi, koneksi ke MCU yang perlu ditentukan adalah untuk pin SER, SRCLK, dan RCLK. Sedangkan untuk pin output, masing-masing pin dihubungkan ke katoda LED. Untuk pin lain, OE dihubungkan ke ground, dan SRCLR dihubungkan ke VCC.
### Nomenclature of the 7400 series IC (opsional)
IC 74HC595 memiliki makna dalam penamaannya. Untuk deskripsi lebih detail Anda dapat meninjau laman Wikipedia berikut: [7400-series Integrated Circuits](https://en.wikipedia.org/wiki/7400-series_integrated_circuits). Berikut adalah ringkasannya:
* Kode 74 di depan merepresentasikan temperatur operasinya: ![TI TTL Prefix](img/74-prefix-nomen.png)
* HC adalah singkatan dari Highspeed CMOS, ini adalah kode family line. Terdapat banyak family line lain seperti L (Low Power), C (CMOS), F (Fast), dan masih banyak lagi. Untuk lebih detailnya silakan baca [di sini](https://en.wikipedia.org/wiki/7400-series_integrated_circuits#Families).
* Sedangkan 595 adalah kode subfamily. Banyak subfamily lain seperti 165 (SIPO SR), 00 (Quad AND Gate), 114 (Dual J-K Flip-Flop), dsb.
IC 7400 series ini didesain oleh perusahaan asal Amerika, Texas Instruments. Namun, perusahaan lain memiliki lisensi untuk memproduksi seri ini. Sehingga, diberikan prefiks untuk memberi keterangan perusahaan yang memproduksinya: ![TTL Manufacturer Prefix](img/74-manprefix-nomen.png)

## Interrupts
### A Brief Description
Interrupt adalah salah satu fungs dasar pada mikrokontroler yang berguna untuk menginterupsi program utama untuk melakukan proses lain yang lebih penting. Secara teknis, interrupt adalah sinyal yang menginformasikan CPU agar menghentikan apapun yang sedang dilakukan untuk sementara waktu. Setelah itu, Interrupt Service Routine (ISR) akan dijalankan. ISR umumnya ditulis dalam bentuk routine/fungsi. Kemudian, CPU akan kembali melanjutkan proses yang sebelumnya diinterupsi.

### Blocking Process
Pertanyaannya, kenapa perlu menggunakan interrupt? Apakah tidak cukup menggunakan if di dalam main loop? Untuk menjawabnya, kita perlu mengingat bahwa sebuah CPU hanya bisa melakukan satu tugas  dalam satu waktu. Sementara itu, proses seperti animasi LED adalah proses yang memakan waktu, sebagian besar waktunya digunakan untuk menunggu beberapa ratus ms agar kita dapat melihat animasinya. 

Misalkan kita menambahkan tombol 'previous', 'next', dan 'restart' yang berguna untuk mengontrol animasi mana yang diputar. Tanpa interrupt, jika salah satu tombol ditekan maka kita harus menunggu animasi saat ini selesai baru menjalankan aksi sesuai tombol.

Dengan interrupt, kita bisa menginterupsi di tengah-tengah proses animasi. Kemudian di dalam ISR, ditentukan animasi mana yang akan dimainkan selanjutnya. Setelah itu, ISR akan kembali ke proses yang diinterupsi. Bedanya, kali ini fungsi tersebut akan dihentikan paksa sehingga iterasi main loop juga akan dihentikan paksa.
### Interrupts in ATMega328P

## Putting it All Together
### The Animation
### Writing The Bytes to Shift Register
### Interrupt for Interface
### Just Watch.

## Sum Up

_*Catatan: Blog pertama pada seri ini ditulis dalam Bahasa Inggris dengan struktur yang tidak konkrit dan grammar yang kacau serta canggung. Blog tersebut juga tidak memberikan contoh kode yang cukup. Saya memutuskan untuk pindah ke Bahasa Indonesia karena target audiens yang lebih mudah diraih adalah audiens lokal. Pembenaran ketiga, bahwasanya jumlah blog teknis yang ditulis dalam Bahasa Indonesia kalah jauh dibandingkan Bahasa Inggris._