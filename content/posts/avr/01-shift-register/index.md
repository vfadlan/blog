---
date: 2025-12-28T05:15:34+07:00
draft: false
params:
  author: Fadlan Abduh
title: Animating LEDs with Shift Register
series: ["AVR"]
series_order: 1
---
## Table of Contents
1. [Intro](#intro)
2. [Shift Register](#shift-register)
3. [IC SN74HC595](#ic-sn74hc595)  
  a. [↳Overview](#overview)  
  b. [↳Pin Descriptions](#pin-descriptions)  
  c. [↳Functional Characteristics](#functional-characteristics)  
  d. [↳Nomenclature of the 7400 series IC (trivia)](#nomenclature-of-the-7400-series-ic-opsional)
4. [Interrupts](#interrupts)  
  a. [↳A Brief Description](#a-brief-description)  
  b. [↳Blocking Process](#blocking-process)  
  c. [↳Interrupts in ATMega328P](#interrupts-in-atmega328p)  
5. [Putting it All Together](#putting-it-all-together)  
  a. [↳Writing Bytes to the Shift Register](#writing-bytes-to-the-shift-register)
  b. [↳The Animation](#the-animation)  
  c. [↳Interrupt for Interface](#interrupt-for-interface)  
  d. [↳Result](#result)
6. [Conclusion](#conclusion)
7. [References and Further Reading](#references-and-further-reading)
8. [Foot Notes](#foot-notes)
## Intro
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

## IC SN74HC595
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
### Nomenclature of the 7400 series IC (trivia)
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

Misalkan kita menambahkan tombol 'previous', 'next', dan 'restart' yang berguna untuk mengontrol animasi mana yang diputar. Tanpa interrupt, jika salah satu tombol ditekan maka tombol itu akan diabakan selama animasi sedang berjalan.

Dengan interrupt, kita bisa menginterupsi di tengah-tengah proses animasi. Kemudian di dalam ISR, ditentukan animasi mana yang akan dimainkan selanjutnya. Setelah itu, ISR akan kembali ke proses yang diinterupsi. Bedanya, kali ini fungsi tersebut akan dihentikan paksa sehingga iterasi main loop juga akan dihentikan paksa.

### Interrupts in ATMega328P
Pada ATMega328P terdapat beberapa macam interrupt, masing-masing memiliki alamat ISR-nya sendiri. 
![Interrupt Vector Table of the ATMega328P](img/atmega328p-intvect.png "Macam sumber interrupt, RESET adalah prioritas tertinggi. Diambil dari datasheet ATMega328P.")
Sebagai contoh, ketika tombol dengan konfigurasi pull-up resistor ditekan, kondisi pin terkait akan berubah. Maka, untuk menerapkan interrupt, digunakan jenis Pin Change Interrupt (PCINT). Terdapat 3 PCINT (PCINT0-PCINT2), PCINT0 serta PCINT 2 masing-masing menangani 8 pin, dan PCINT1 menangani 7 pin.

Untuk menggunakan fitur interrupt, sertakan library `<avr/interrupt.h>`. Secara default interrupt tidak aktif, untuk mengaktifkan interrupt global, pangggil fungsi `sei()`. Selanjutnya, untuk mengaktifkan interrupt vector spesifik, register interrupt terkait perlu dimanipulasi.

![EIMSK Register](img/eimsk.png)
```c
EIMSK |= (1 << INT1);
sei();
```
Ekspresi di atas akan memodifikasi EIMSK (External Interrupt Mask Register), mengubah 0 menjadi 1 pada bit INT1. Sehingga interrupt vector INT1 menjadi aktif. Pemanggilan fungsi `sei()` akan mengaktifkan interrupt secara global. 

Ketika terdapat sinyal interrupt INT1 diciptakan, MCU akan segera menjalankan ISR yang sudah didefinisikan:
```c
ISR (INT1_vect) {
  // isr tasks
}
```
ISR dapat diinterupsi oleh interrupt vector yang memiliki prioritas lebih tinggi. Contoh, INT1 dapat diinterupsi oleh INT0 dan RESET.

Proses di dalam ISR sebaiknya dibuat seringkas mungkin. Salah satu cara untuk mempersingkatnya adalah menggunakan variable sebagai flag. Ketika sinyal interrupt diberikan, ISR mengisi flag dengan nilai `true`. Flag tersebut akan dievaluasi menggunakan control-flow sebelum mengeksekusi suatu proses.
## Putting it All Together
### Writing Bytes to the Shift Register
Sebuah LED dapat direpresentasikan sebagai satu bit, delapan LED direpresentasikan menjadi delapan bit atau satu byte. Delapan LED tersebut selanjutnya dihubungkan ke output parallel IC shift register, SN74HC595. Sekarang, kita perlu mencari cara untuk mengirimkan data secara serial ke dalam IC tersebut.

Dengan mengabaikan pin-pin output dan pin lain yang terhubung ke ground atau VCC, berikut adalah pin yang tersisa dan perlu dihubungkan ke ATMega328P:
1. SRCLK (11) ke PD0 (2),
2. RCLK (12) ke PD1 (3), dan
3. SER (14) ke PD2 (4).

Pin-pin di atas digunakan dalam fungsi-fungsi primer yang melakukan: bit shifting, trigger SR clock, latch SR ke output.

_*Catatan: Nomor pin SN74HC595 mengacu package DIP-16, dan ATMega328P DIP-28_
```c
#define SER PD0
#define R_CLK PD1
#define SR_CLK PD2

void tick(void) {
  PORTD |= (1 << SR_CLK);
  _delay_ms(1);
  PORTD &= (~(1 << SR_CLK));
}
```
Ketika routine `tick()` dieksekusi, data shift register akan digeser dengan nilai bit ke-0 mengikuti nilai `SER` atau `PD0` pada saat fungsi dieksekusi. Hal ini sesuai dengan karakteristik SR pada datasheet yang [sudah kita bahas](#functional-characteristics).
```c
void shift_bit(bool bit) {
  if (bit)    PORTD |= (1 << SER);
  else        PORTD &= (~(1 << SER));

  tick();
}
```
Fungsi `shift_bit(bool bit)` menerima boolean sebagai parameter. Boolean tersebut dijadikan sebagai nilai `PD0`. Setelah nilai `PD0` diubah, fungsi `tick()` dipanggil sehingga bit ke-0 pada SR ditentukan oleh parameter `bit`.
```c
bool latch(void) {
  if (restart_requested) {
    restart_requested = false;
    return false;
  }

  PORTD |= (1 << R_CLK);
  _delay_ms(1);
  PORTD &= (~(1 << R_CLK));
  return true;
}
```
Fokus ke bawah terlebih dahulu, fungsi `latch()` akan menyalakan `R_CLK` atau `PD1` selama 1ms. Sesuai datasheet, high pada `RCLK` akan menyalin data pada SR ke storage register yang menyimpan nilai output.

Kembali ke atas. Di dalam fungsi `latch()`, `restart_requested` yang merupakan variable flag akan diperiksa. Jika bernilai `true`, flag tersebut akan dikembalikan ke nilai defaultnya. Setelah itu menghentikan eksekusi fungsi `latch()` dengan `return false;`.

Proses serupa terjadi secara bertumpuk/bersarang, fungsi yang memanggil `latch()` dan mendapatkan nilai `false` akan mengakhiri dirinya dengan mengembalikan nilai `false`. Dan seterusnya.

Sehingga ketika sinyal interrupt diberikan, semua proses secara instan akan terhenti dan memulai ulang _main-loop_ untuk memulai animasi sebelumnya, selanjutnya, atau memulai ulang animasi saat ini.

```c
bool sr_write(uint8_t data) {
  for (int i=0; i<8; i++) {
    shift_bit(data & 0x80);
    data <<= 1;
  }
  return latch();
}
```
Fungsi `sr_write(uint8_t data)` menerima parameter `data` yang berukuran 8-bit. Fungsi tersebut memasukkan data bit-demi-bit ke dalam SR dimulai dari _Most Significant Bit_ (MSB First).

Di dalam setiap iterasi for, bit paling kiri (`data & 0x80`) dimasukkan ke dalam SR. Selanjutnya, variabel `data` akan digeser 1 posisi ke kiri.

Setelah semua bit dimasukkan ke dalam SR, fungsi `latch() dipanggil.
### The Animation
### Interrupt for Interface
```c
#include <avr/interrupt.h>
int main() {
  PCICR |= (1 << PCIE2);
  PCMSK2 |=  (1 << PCINT21) | (1 << PCINT22) | (1 << PCINT23);
  sei();

  while (1) {
    // main loop program here
  }
}
```
Untuk mendefinisikan program dengan interrupt, diperlukan library `<avr/interrupt.h>`. Selanjutnya, terdapat 2 register yang perlu ditangani:
* `PCICR` (Pin Change Interrupt Control Register)


### Result

## Conclusion
## References and Further Reading