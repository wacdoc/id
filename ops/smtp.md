# Bangun server pengiriman email SMTP Anda sendiri

## pembukaan

SMTP dapat langsung membeli layanan dari vendor cloud, seperti:

* [SMTP Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Dorongan email cloud Ali](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Anda juga dapat membangun server email Anda sendiri - pengiriman tidak terbatas, biaya keseluruhan rendah.

Di bawah ini, kami mendemonstrasikan langkah demi langkah cara membangun server email kami sendiri.

## Pemilihan server

Server SMTP yang dihosting sendiri memerlukan IP publik dengan port 25, 456, dan 587 terbuka.

Cloud publik yang umum digunakan telah memblokir port ini secara default, dan dimungkinkan untuk membukanya dengan mengeluarkan perintah kerja, tetapi bagaimanapun juga sangat merepotkan.

Saya merekomendasikan membeli dari host yang membuka port ini dan mendukung pengaturan nama domain terbalik.

Di sini, saya merekomendasikan [Contabo](https://contabo.com) .

Contabo adalah penyedia hosting yang berbasis di Munich, Jerman, didirikan pada tahun 2003 dengan harga yang sangat kompetitif.

Jika Anda memilih Euro sebagai mata uang pembelian, harganya akan lebih murah (server dengan memori 8GB dan 4 CPU harganya sekitar 529 yuan per tahun, dan biaya pemasangan awal gratis selama satu tahun).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Saat melakukan pemesanan, beri komentar `prefer AMD` , dan server dengan CPU AMD akan memiliki kinerja yang lebih baik.

Berikut ini, saya akan menggunakan VPS Contabo sebagai contoh untuk mendemonstrasikan cara membangun server email Anda sendiri.

## konfigurasi sistem Ubuntu

Sistem operasi di sini adalah Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Jika server di ssh menampilkan `Welcome to TinyCore 13!` (seperti yang ditunjukkan pada gambar di bawah), artinya sistem belum diinstal. Putuskan sambungan ssh dan tunggu beberapa menit untuk masuk lagi.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Ketika `Welcome to Ubuntu 22.04.1 LTS` muncul, inisialisasi selesai, dan Anda dapat melanjutkan dengan langkah-langkah berikut.

### [Opsional] Inisialisasi lingkungan pengembangan

Langkah ini opsional.

Untuk kenyamanan, saya meletakkan instalasi dan konfigurasi sistem perangkat lunak ubuntu di [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Jalankan perintah berikut untuk menginstal dengan satu klik.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Pengguna Cina, silakan gunakan perintah berikut, dan bahasa, zona waktu, dll. Akan diatur secara otomatis.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo mengaktifkan IPV6

Aktifkan IPV6 agar SMTP juga dapat mengirim email dengan alamat IPV6.

edit `/etc/sysctl.conf`

Ubah atau tambahkan baris berikut

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Tindak lanjuti dengan [tutorial contabo: Menambahkan konektivitas IPv6 ke server Anda](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Edit `/etc/netplan/01-netcfg.yaml` , tambahkan beberapa baris seperti yang ditunjukkan pada gambar di bawah ini (File konfigurasi default Contabo VPS sudah memiliki baris ini, cukup batalkan komentarnya).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Kemudian `netplan apply` untuk membuat konfigurasi yang dimodifikasi berlaku.

Setelah konfigurasi berhasil, Anda dapat menggunakan `curl 6.ipw.cn` untuk melihat alamat ipv6 jaringan eksternal Anda.

## Klon operasi repositori konfigurasi

```
git clone https://github.com/wactax/ops.soft.git
```

## Hasilkan sertifikat SSL gratis untuk nama domain Anda

Mengirim email memerlukan sertifikat SSL untuk enkripsi dan penandatanganan.

Kami menggunakan [acme.sh](https://github.com/acmesh-official/acme.sh) untuk menghasilkan sertifikat.

acme.sh adalah alat penandatanganan sertifikat otomatis sumber terbuka,

Masuk ke gudang konfigurasi ops.soft, jalankan `./ssl.sh` , dan folder `conf` akan dibuat di **direktori atas** .

Temukan penyedia DNS Anda dari [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , edit `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Kemudian jalankan `./ssl.sh 123.com` untuk menghasilkan sertifikat `123.com` dan `*.123.com` untuk nama domain Anda.

Proses pertama akan menginstal [acme.sh](https://github.com/acmesh-official/acme.sh) secara otomatis dan menambahkan tugas terjadwal untuk perpanjangan otomatis. Anda dapat melihat `crontab -l` , ada baris seperti berikut.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Jalur untuk sertifikat yang dihasilkan adalah seperti `/mnt/www/.acme.sh/123.com_ecc。`

Pembaruan sertifikat akan memanggil skrip `conf/reload/123.com.sh` , edit skrip ini, Anda dapat menambahkan perintah seperti `nginx -s reload` untuk menyegarkan cache sertifikat dari aplikasi terkait.

## Bangun server SMTP dengan chasquid

[chasquid](https://github.com/albertito/chasquid) adalah server SMTP sumber terbuka yang ditulis dalam bahasa Go.

Sebagai pengganti program mail server kuno seperti Postfix dan Sendmail, chasquid lebih sederhana dan mudah digunakan, dan juga lebih mudah untuk pengembangan sekunder.

Jalankan `./chasquid/init.sh 123.com` akan diinstal secara otomatis dengan satu klik (ganti 123.com dengan nama domain pengirim Anda).

## Konfigurasi DKIM Tanda Tangan Email

DKIM digunakan untuk mengirim tanda tangan email untuk mencegah surat diperlakukan sebagai spam.

Setelah perintah berhasil dijalankan, Anda akan diminta untuk menyetel data DKIM (seperti gambar di bawah).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Cukup tambahkan catatan TXT ke DNS Anda (seperti yang ditunjukkan di bawah).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Lihat status & log layanan

 `systemctl status chasquid` Lihat status layanan.

Keadaan operasi normal seperti yang ditunjukkan pada gambar di bawah ini

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` atau `journalctl -xeu chasquid` dapat melihat log kesalahan.

## Membalikkan konfigurasi nama domain

Nama domain terbalik memungkinkan alamat IP diselesaikan ke nama domain yang sesuai.

Menyetel nama domain terbalik dapat mencegah email diidentifikasi sebagai spam.

Saat email diterima, server penerima akan melakukan analisis nama domain balik pada alamat IP server pengirim untuk memastikan apakah server pengirim memiliki nama domain balik yang valid.

Jika server pengirim tidak memiliki nama domain terbalik atau jika nama domain terbalik tidak cocok dengan alamat IP server pengirim, server penerima dapat mengenali email tersebut sebagai spam atau menolaknya.

Kunjungi [https://my.contabo.com/rdns](https://my.contabo.com/rdns) dan konfigurasikan seperti yang ditunjukkan di bawah ini

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Setelah menetapkan nama domain terbalik, ingatlah untuk mengonfigurasi resolusi penerusan nama domain ipv4 dan ipv6 ke server.

## Edit nama host chasquid.conf

Ubah `conf/chasquid/chasquid.conf` ke nilai nama domain terbalik.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Kemudian jalankan `systemctl restart chasquid` untuk me-restart layanan.

## Cadangkan conf ke repositori git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Misalnya, saya mencadangkan folder conf ke proses github saya sendiri sebagai berikut

Buat gudang pribadi terlebih dahulu

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Masuk ke direktori conf dan kirim ke gudang

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Tambahkan pengirim

berlari

```
chasquid-util user-add i@wac.tax
```

Bisa tambah pengirim

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Verifikasi bahwa kata sandi diatur dengan benar

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Setelah menambahkan pengguna, `chasquid/domains/wac.tax/users` akan diperbarui, ingat untuk mengirimkannya ke gudang.

## DNS menambahkan catatan SPF

SPF (Sender Policy Framework) adalah teknologi verifikasi email yang digunakan untuk mencegah penipuan email.

Itu memverifikasi identitas pengirim email dengan memeriksa bahwa alamat IP pengirim cocok dengan catatan DNS dari nama domain yang diklaimnya, mencegah penipu mengirim email palsu.

Menambahkan catatan SPF dapat mencegah email diidentifikasi sebagai spam sebanyak mungkin.

Jika server nama domain Anda tidak mendukung jenis SPF, tambahkan saja catatan jenis TXT.

Misalnya, SPF dari `wac.tax` adalah sebagai berikut

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF untuk `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Perhatikan bahwa saya `include:_spf.google.com` di sini, ini karena saya akan mengonfigurasi `i@wac.tax` sebagai alamat pengirim di kotak surat Google nanti.

## Konfigurasi DNS DMARC

DMARC adalah singkatan dari (Domain-based Message Authentication, Reporting & Conformance).

Ini digunakan untuk menangkap pantulan SPF (mungkin disebabkan oleh kesalahan konfigurasi, atau orang lain berpura-pura menjadi Anda untuk mengirim spam).

Tambahkan catatan TXT `_dmarc` ,

Isinya adalah sebagai berikut

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Arti dari masing-masing parameter adalah sebagai berikut

### p (Kebijakan)

Menunjukkan cara menangani email yang gagal dalam verifikasi SPF (Sender Policy Framework) atau DKIM (DomainKeys Identified Mail). Parameter p dapat diatur ke salah satu dari tiga nilai:

* none: Tidak ada tindakan yang diambil, hanya hasil verifikasi yang dikembalikan ke pengirim melalui mekanisme pelaporan email.
* Karantina: Masukkan email yang belum lolos verifikasi ke dalam folder spam, tetapi tidak akan langsung menolak email tersebut.
* tolak: Menolak langsung email yang gagal verifikasi.

### fo (Opsi Kegagalan)

Menentukan jumlah informasi yang dikembalikan oleh mekanisme pelaporan. Itu dapat diatur ke salah satu dari nilai berikut:

* 0: Laporkan hasil validasi untuk semua pesan
* 1: Hanya laporkan pesan yang gagal verifikasi
* d: Hanya laporkan kegagalan verifikasi nama domain
* s: hanya laporkan kegagalan verifikasi SPF
* l: Hanya laporkan kegagalan verifikasi DKIM

### rua & ruf

* rua (URI Pelaporan untuk laporan Gabungan): Alamat email untuk menerima laporan gabungan
* ruf (Pelaporan URI untuk laporan Forensik): alamat email untuk menerima laporan terperinci

## Tambahkan data MX untuk meneruskan email ke Google Mail

Karena saya tidak dapat menemukan kotak surat perusahaan gratis yang mendukung alamat universal (Catch-All, dapat menerima email apa pun yang dikirim ke nama domain ini, tanpa batasan awalan), saya menggunakan chasquid untuk meneruskan semua email ke kotak surat Gmail saya.

**Jika Anda memiliki kotak surat bisnis berbayar sendiri, harap jangan mengubah MX dan lewati langkah ini.**

Edit `conf/chasquid/domains/wac.tax/aliases` , setel kotak surat penerusan

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` menunjukkan semua email, `i` adalah awalan alamat email dari pengguna pengirim yang dibuat di atas. Untuk meneruskan email, setiap pengguna perlu menambahkan baris.

Kemudian tambahkan data MX (saya arahkan langsung ke alamat nama domain terbalik di sini, seperti yang ditunjukkan pada baris pertama pada gambar di bawah).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Setelah konfigurasi selesai, Anda dapat menggunakan alamat email lain untuk mengirim email ke `i@wac.tax` dan `any123@wac.tax` untuk mengetahui apakah Anda dapat menerima email di Gmail.

Jika tidak, periksa log chasquid ( `grep chasquid /var/log/syslog` ).

## Kirim email ke i@wac.tax dengan Google Mail

Setelah Google Mail menerima surat tersebut, tentu saja saya berharap membalas dengan `i@wac.tax` alih-alih i.wac.tax@gmail.com.

Kunjungi [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) dan klik "Tambahkan alamat email lain".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Kemudian, masukkan kode verifikasi yang diterima oleh email yang diteruskan.

Terakhir, ini dapat disetel sebagai alamat pengirim default (bersama dengan opsi untuk membalas dengan alamat yang sama).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Dengan cara ini, kami telah menyelesaikan pembuatan server email SMTP dan pada saat yang sama menggunakan Google Mail untuk mengirim dan menerima email.

## Kirim email percobaan untuk memeriksa apakah konfigurasi berhasil

Masukkan `ops/chasquid`

Jalankan `direnv allow` untuk menginstal dependensi (direnv telah diinstal pada proses inisialisasi satu kunci sebelumnya dan sebuah kait telah ditambahkan ke shell)

lalu lari

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Arti dari parameter adalah sebagai berikut

* pengguna: nama pengguna SMTP
* lulus: kata sandi SMTP
* kepada: penerima

Anda dapat mengirim email percobaan.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Disarankan untuk menggunakan Gmail untuk menerima email percobaan untuk memeriksa apakah konfigurasi berhasil.

### Enkripsi standar TLS

Seperti yang ditunjukkan pada gambar di bawah, ada kunci kecil ini, yang berarti sertifikat SSL telah berhasil diaktifkan.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Kemudian klik "Tampilkan Email Asli"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Seperti yang ditunjukkan pada gambar di bawah ini, halaman surat asli Gmail menampilkan DKIM, yang berarti konfigurasi DKIM berhasil.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Periksa Diterima di header email asli, dan Anda dapat melihat bahwa alamat pengirimnya adalah IPV6, yang berarti IPV6 juga berhasil dikonfigurasi.
