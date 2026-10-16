# Pra-Praktikum Modul 2
## HTTP Server, DNS Server, dan Reverse Proxy

**Pre-requisite**

- GNS3 VM dan GNS3-GUI sudah terinstall.
- Appliance `netics-pc` / `netics-server` dan `netics-pc-desktop`.

**Cakupan modul:** HTTP Server (nginx), DNS Server (BIND), Integrasi DNS dengan DHCP, Reverse Proxy berbasis path, Reverse Proxy multi-backend lintas port.

> **Catatan:** Semua problem dan bagian wajib dikerjakan sebagai persiapan praktikum hari H. Pengerjaan laporan mengikuti instruksi di README GitHub. `[IP Prefix]` disesuaikan dengan pembagian yang sudah diumumkan (contoh: `192.168`, sehingga `[IP Prefix].200.11` menjadi `192.168.200.11`). **Ganti semua `[IP Prefix]` sebelum menyalin perintah.**

---

## Daftar Isi

0. [Pemetaan Node](#pemetaan-node)
1. [Bagian 0 - Persiapan (Router, NAT, DHCP)](#0-pre---persiapan)
2. [Bagian A - HTTP Server](#a-bagian-a---http-server)
3. [Bagian B - DNS Server](#b-bagian-b---dns-server)
4. [Bagian C - Reverse Proxy](#c-bagian-c---reverse-proxy)
5. [Verifikasi Menyeluruh](#e-verifikasi-menyeluruh)
6. [Tugas Mandiri Bagian C](#f-tugas-mandiri-bagian-c)
7. [Catatan Koreksi terhadap Soal Asli](#catatan-koreksi-terhadap-soal-asli)
8. [Cheat Sheet Debugging](#cheat-sheet-debugging)

---

## Pemetaan Node

| Node | Appliance | Peran | IP Address |
|---|---|---|---|
| Router | netics-pc | Menyediakan akses internet dari NAT | eth0: Auto (DHCP); eth1: `[IP Prefix].200.1/24` (statis) |
| Sargon, Iberia, Higashi | netics-pc | Client uji command line (curl) | Auto (DHCP) |
| Ognisko | netics-pc-desktop | Client uji browser (VNC + Chromium/Firefox) | Auto (DHCP) |
| Aegir | netics-pc | DHCP Server | `[IP Prefix].200.10/24` (statis), gateway `[IP Prefix].200.1`, pool `[IP Prefix].200.100-150`; tambahkan `option router [IP Prefix].200.1` di dhcpd.conf |
| Kazdel | netics-pc / netics-server | Web Server (Bagian A) → Reverse Proxy (Bagian C) | `[IP Prefix].200.11/24` (statis), gateway `[IP Prefix].200.1` |
| Chernobog | netics-pc / netics-server | DNS Server (BIND) | `[IP Prefix].200.12/24` (statis), gateway `[IP Prefix].200.1` |
| web1 | netics-pc / netics-server | Backend HTTP server | `[IP Prefix].200.20/24` (statis), port 8080, gateway `[IP Prefix].200.1` |
| web2 | netics-pc / netics-server | Backend HTTP server + aplikasi Python | `[IP Prefix].200.21/24` (statis), port 8080 & 5000, gateway `[IP Prefix].200.1` |
| web3 *(Tugas Mandiri)* | netics-pc / netics-server | Backend HTTP server | `[IP Prefix].200.22/24` (statis), port 8080, gateway `[IP Prefix].200.1` |

### Teori: mengapa ada IP statis dan IP DHCP?

- **Server** (Kazdel, Chernobog, web1, web2, Aegir) memakai **IP statis** karena alamatnya harus tetap dan dapat diprediksi. DNS record, konfigurasi reverse proxy, dan opsi DHCP semuanya menunjuk ke IP ini. Jika IP server berubah, semua yang merujuknya rusak.
- **Client** memakai **DHCP** karena IP mereka tidak perlu diketahui siapa pun; yang penting mereka mendapat IP, gateway, dan DNS secara otomatis.
- **Aegir (DHCP server) harus statis** karena ia tidak bisa meminta IP dari dirinya sendiri (masalah *chicken-and-egg*).

---

## 0. Pre - Persiapan

**Topologi:** `NAT1 ── Router (eth0)`; `Router (eth1) ── Switch1`.

### Langkah 0.1 - Konfigurasi Router dan hubungkan eth0 ke NAT

Konfigurasi interface Router (via menu *Configure → Edit network configuration* pada node GNS3):

```
# eth0 : DHCP dari NAT
auto eth0
iface eth0 inet dhcp

# eth1 : sisi LAN internal, statis
auto eth1
iface eth1 inet static
        address [IP Prefix].200.1
        netmask 255.255.255.0
```

**Teori**

- Router punya dua kaki: `eth0` menghadap **luar** (NAT GNS3, mendapat IP via DHCP dari NAT gateway) dan `eth1` menghadap **dalam** (LAN lab). `eth1` harus statis karena IP-nya (`.200.1`) dipakai sebagai *default gateway* oleh semua node lain.
- Node NAT di GNS3 berfungsi seperti modem rumahan: menyediakan DHCP dan akses ke internet.

**Debug**

```bash
ip addr show eth0        # harus ada IP dari NAT (mis. 192.168.124.x)
ip addr show eth1        # harus [IP Prefix].200.1/24
ip route                 # harus ada "default via <gateway NAT> dev eth0"
ip link set eth1 up      # jika eth1 DOWN
```

| Gejala | Kemungkinan penyebab |
|---|---|
| eth0 tidak dapat IP | Kabel belum tersambung ke NAT, atau node belum selesai boot (tunggu `udhcpc: lease obtained`) |
| eth1 tidak punya IP | File interfaces belum disimpan / node belum direstart |
| Tidak ada default route | NAT tidak mengirim gateway; cek lease DHCP di eth0 |

### Langkah 0.2 - Pastikan Router mendapat IP otomatis

Masuk console Router. Harus muncul log seperti:

```
udhcpc: broadcasting discover
udhcpc: lease of 192.168.124.189 obtained from 192.168.124.1, lease time 3600
```

**Teori:** `udhcpc` adalah DHCP *client* bawaan BusyBox/Alpine. Urutan DHCP: **D**iscover → **O**ffer → **R**equest → **A**cknowledge (DORA). "lease obtained" artinya DORA selesai.

**Debug**

```bash
ip addr show eth0
udhcpc -i eth0 -n        # paksa minta ulang lease (-n: keluar jika gagal)
```

### Langkah 0.3 - Pasang iptables dan aktifkan NAT (MASQUERADE)

```bash
apk update
apk add iptables
iptables -t nat -A POSTROUTING -j MASQUERADE -o eth0 -s [IP Prefix].0.0/16
```

**Teori**

| Komponen | Fungsi |
|---|---|
| `apk update` | Mengunduh indeks paket terbaru dari repositori Alpine. Tanpanya `apk add` tidak tahu paket apa yang tersedia. |
| `apk add iptables` | Memasang tool pengatur firewall/NAT Linux (frontend untuk netfilter di kernel). |
| `-t nat` | Memakai tabel **nat** (tabel khusus untuk penerjemahan alamat). |
| `-A POSTROUTING` | Menambahkan aturan ke chain POSTROUTING: dievaluasi **setelah** keputusan routing, tepat sebelum paket keluar interface. |
| `-o eth0` | Hanya paket yang keluar lewat eth0 (sisi internet). |
| `-s [IP Prefix].0.0/16` | Hanya paket dengan sumber dari jaringan lab. |
| `-j MASQUERADE` | Mengganti IP sumber paket dengan IP milik eth0 saat itu. |

**Mengapa NAT dibutuhkan?** IP lab (`[IP Prefix].200.x`) adalah IP **privat**; router di internet tidak tahu jalur balik ke IP tersebut. Dengan MASQUERADE, semua paket tampak berasal dari IP eth0 Router, dan balasan dikembalikan ke node asal berkat tabel *connection tracking* (`conntrack`). MASQUERADE dipilih (bukan SNAT) karena IP eth0 didapat dari DHCP dan bisa berubah.

**Syarat tambahan agar routing jalan: IP forwarding.** Tanpanya kernel membuang paket yang bukan tujuan dirinya.

```bash
sysctl net.ipv4.ip_forward           # harus = 1
sysctl -w net.ipv4.ip_forward=1      # jika 0
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```

**Debug**

```bash
iptables -t nat -L POSTROUTING -n -v --line-numbers   # rule ada? kolom pkts > 0 setelah ada traffic?
iptables -t nat -S POSTROUTING                        # tampilan ringkas
iptables -L FORWARD -n -v                             # policy FORWARD DROP? (bisa memblokir)
ip route get 8.8.8.8                                  # pastikan interface keluar benar-benar eth0
tcpdump -ni eth0 icmp                                 # source IP harus sudah berubah ke IP eth0
conntrack -L                                          # tabel NAT aktif (apk add conntrack-tools)
watch -n1 'iptables -t nat -L POSTROUTING -n -v'      # counter naik realtime
```

Jika FORWARD berpolicy DROP:

```bash
iptables -A FORWARD -s [IP Prefix].0.0/16 -o eth0 -j ACCEPT
iptables -A FORWARD -d [IP Prefix].0.0/16 -i eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

| Gejala | Penyebab / solusi |
|---|---|
| Counter rule = 0 | Tidak ada traffic yang lewat: cek gateway client, `ip_forward`, atau nama interface salah (`eth0` vs `ens3`) |
| Counter naik tapi tidak ada balasan | FORWARD chain memblokir, atau firewall upstream |
| Rule hilang setelah reboot | iptables tidak persisten. Di Alpine: `rc-update add iptables` lalu `/etc/init.d/iptables save`. Karena node GNS3 biasanya disetel lewat *start command*, letakkan perintah ini di start script node |

### Langkah 0.4 - Ping google.com dari Router

```bash
ping -c 3 google.com
```

**Teori:** Tes ini memverifikasi **dua hal sekaligus**: (1) konektivitas IP ke internet, dan (2) resolusi DNS (nama → IP).

**Debug (urutan dari lapisan bawah ke atas)**

```bash
ping -c 3 127.0.0.1          # stack TCP/IP lokal OK?
ping -c 3 192.168.124.1      # gateway NAT (sesuaikan) OK?   -> masalah L2/L3 lokal
ping -c 3 8.8.8.8            # internet OK tanpa DNS?        -> masalah NAT/routing upstream
ping -c 3 google.com         # DNS OK?
cat /etc/resolv.conf         # ada nameserver?
```

- Ping IP berhasil, tapi nama gagal → masalah **DNS** (lanjut ke langkah 0.5).
- Ping 8.8.8.8 gagal → masalah **routing/NAT**.

### Langkah 0.5 - Set DNS sementara di Aegir (DHCP server)

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
ping -c 3 google.com
```

**Teori:** `/etc/resolv.conf` adalah file yang dibaca *resolver* Linux untuk mengetahui server DNS mana yang ditanya. Aegir bukan client DHCP (IP statis), jadi tidak ada yang mengisi file ini otomatis. Kita isi manual dengan DNS publik Google agar `apk update` bisa me-resolve nama repositori.

> Isi `resolv.conf` hilang bila node direstart; tulis ulang setiap kali sebelum `apk update`.

**Debug**

```bash
cat /etc/resolv.conf
ping -c 2 8.8.8.8            # jika gagal: cek gateway di Aegir (ip route)
ip route                     # harus ada default via [IP Prefix].200.1
```

Jika Aegir tidak punya default route, tambahkan `gateway [IP Prefix].200.1` di interfaces-nya.

### Langkah 0.6 - Konfigurasi DHCP Server (Aegir)

Pasang dan konfigurasikan seperti Modul 1. Contoh `dhcpd.conf` yang ekuivalen dengan ketentuan soal (nama opsi mengikuti sintaks ISC dhcpd):

```
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet [IP Prefix].200.0 netmask 255.255.255.0 {
    range [IP Prefix].200.100 [IP Prefix].200.150;
    option routers [IP Prefix].200.1;
    # Ditambahkan di Bagian B (setelah DNS server siap):
    # option domain-name-servers [IP Prefix].200.12, 8.8.8.8;
    # option domain-name "netics.my.id";
}
```

> Soal menulis `option router ...`. Nama baku di ISC dhcpd adalah `option routers`. Jika dhcpd menolak (`unknown option`), gunakan `routers`; jika Modul 1 Anda mendefinisikan alias sendiri, ikuti Modul 1.

**Teori**

| Direktif | Fungsi |
|---|---|
| `subnet ... netmask ...` | Mendefinisikan jaringan yang dilayani. **Wajib** cocok dengan jaringan interface Aegir, jika tidak dhcpd menolak menyala. |
| `range` | Pool IP yang boleh dibagikan (`.100-.150`). Server statis (`.10-.22`) di luar pool agar tidak bentrok. |
| `option routers` | Mengisi *default gateway* client. Tanpa ini client dapat IP tetapi tak bisa keluar jaringan lokal. |
| `lease time` | Lama IP dipinjamkan sebelum client harus memperbarui. |

**Debug**

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf      # cek sintaks (jangan start dulu)
dhcpd -4 -f -d eth0                    # jalankan foreground + debug, lihat DORA di layar
cat /var/lib/dhcp/dhcpd.leases         # daftar lease yang sudah diberikan
```

Di sisi client:

```bash
ip addr show eth0
ip route                               # default via [IP Prefix].200.1 ?
udhcpc -i eth0                         # minta ulang lease
```

| Gejala | Penyebab |
|---|---|
| `No subnet declaration for eth0` | Subnet di dhcpd.conf tidak cocok dengan IP eth0 Aegir |
| Client tidak dapat IP | Aegir belum jalan, atau client dan Aegir berbeda segmen switch |
| Client dapat IP tapi tak ada default route | `option routers` belum ditulis / typo |
| Perubahan config tidak berlaku | dhcpd belum di-restart / client belum meminta ulang lease |

---

## A. Bagian A - HTTP Server

### Tujuan Pembelajaran

1. Memahami HTTP server sebagai contoh konkret aplikasi jaringan di **Application Layer**.
2. Memasang dan menjalankan web server **nginx** untuk *information sharing* sederhana.

### Tools

- GNS3 appliance `netics-pc` dan `netics-pc-desktop`
- nginx
- `curl` (di Sargon) dan browser (di Ognisko)

### Skenario Topologi

Satu switch menghubungkan Sargon, Iberia, Higashi, Ognisko (semua DHCP), Aegir (DHCP server, `.200.10`), dan Kazdel (web server statis, `.200.11`). Satu node Router (netics-pc) menyediakan internet melalui NAT.

### Problem A.1 - Setup HTTP Server

**Soal:** Setup web server nginx pada Kazdel dengan IP `[IP Prefix].200.11/24`.

**Konvensi direktori**

| Direktori | Fungsi |
|---|---|
| `/root/myconfig` | File konfigurasi (nginx.conf, start script) |
| `/root/myweb` | File web (index.html) |
| `/root/mylogs` | Pencatatan log |

**Teori:** Memisahkan config, konten, dan log ke tiga direktori mempermudah backup, debugging (log selalu di satu tempat), dan memudahkan direktori ini dikelola terpisah dari lokasi default `/etc/nginx` dan `/var/www`.

---

#### Langkah A.1.1 - Set IP statis Kazdel lewat konfigurasi interface

Edit interfaces (GNS3: klik kanan node → *Configure* → *Edit network configuration*), uncomment dan sesuaikan bagian static:

```
# Static config for eth0
auto eth0
iface eth0 inet static
        address [IP Prefix].200.11
        netmask 255.255.255.0
        gateway [IP Prefix].200.1
```

Pastikan blok **DHCP config** untuk eth0 di-*comment* agar tidak bentrok.

**Teori:** `ip addr add` hanya mengubah state kernel yang hilang saat reboot. Mengedit file interfaces membuat IP **persisten**. `gateway` memberi default route ke Router supaya Kazdel bisa mengakses internet (untuk `apk add`) dan membalas client di luar subnet.

**Debug**

```bash
ip addr show eth0          # IP = [IP Prefix].200.11/24 ?
ip route                   # default via [IP Prefix].200.1 ?
ping -c 2 [IP Prefix].200.1
ping -c 2 [IP Prefix].200.10
```

| Gejala | Penyebab |
|---|---|
| IP masih DHCP / IP tidak berubah | Node belum direstart setelah edit |
| Tidak ada default route | Baris `gateway` tertinggal ter-comment |
| Ping Router gagal | Salah subnet/netmask atau kabel ke switch salah port |

---

#### Langkah A.1.2 - Install nginx

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apk update
apk add nginx
```

**Teori:** Kazdel statis sehingga tak ada DHCP yang mengisi DNS; kita isi manual agar `apk` dapat me-resolve repositori. nginx adalah web server berperforma tinggi yang juga sering dipakai sebagai reverse proxy (Bagian C), jadi satu software dipakai ulang di seluruh modul.

**Debug**

```bash
ping -c 2 8.8.8.8 && ping -c 2 dl-cdn.alpinelinux.org
nginx -v                   # berhasil terpasang?
which nginx
```

Jika `apk update` gagal → cek resolv.conf, default route, dan NAT di Router (Langkah 0.3).

---

#### Langkah A.1.3 - Buat struktur direktori

```bash
mkdir -p /root/myconfig
mkdir -p /root/myweb
mkdir -p /root/mylogs
```

**Teori:** `-p` membuat direktori induk bila belum ada dan tidak error jika sudah ada (aman diulang/*idempotent*). nginx tidak otomatis membuat folder log; jika `/root/mylogs` tidak ada, nginx **gagal start** saat membuka `error_log`.

**Debug:** `ls -ld /root/myconfig /root/myweb /root/mylogs`

---

#### Langkah A.1.4 - Buat `index.html` di `/root/myweb`

```html
<html>
<head>
<title>This is My Web</title>
</head>
<body>
<p>
This is my web
</p>
</body>
</html>
```

**Teori:** HTTP server pada dasarnya memetakan **URL → file**. Saat client meminta `/`, nginx mencari file `index` di direktori `root`. File ini adalah konten yang dikirim sebagai body respons.

**Debug:** `cat /root/myweb/index.html`; periksa permission: `ls -l /root/myweb/` (file harus bisa dibaca user nginx; di sini `user root`).

---

#### Langkah A.1.5 - Buat `nginx.conf` di `/root/myconfig`

```nginx
user root;
worker_processes auto;
worker_cpu_affinity auto;
pid /tmp/nginx.pid;
error_log /root/mylogs/error.log;

events { worker_connections 768; }

http {
    server {
        listen 80;
        server_name _;
        root /root/myweb;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

**Teori: arti tiap direktif**

| Direktif | Penjelasan |
|---|---|
| `user root` | Worker process berjalan sebagai root agar bisa membaca `/root/myweb` (direktori di home root biasanya tertutup untuk user lain). Untuk lab ini boleh; di produksi sebaiknya user non-privileged. |
| `worker_processes auto` | Jumlah worker = jumlah core CPU. |
| `worker_cpu_affinity auto` | Mengikat worker ke core tertentu. |
| `pid /tmp/nginx.pid` | Lokasi file PID; dipindah ke `/tmp` agar selalu writable. |
| `error_log` | File log error (pusat debugging). |
| `events { worker_connections }` | Koneksi simultan maksimum per worker. Blok `events` **wajib ada** meski kosong. |
| `http { server { ... } }` | Blok konfigurasi HTTP; `server` = satu *virtual host*. |
| `listen 80` | Port TCP standar HTTP. |
| `server_name _` | Penampung "nama apa saja" (*catch-all*). |
| `root`, `index` | Folder konten dan nama file default. |
| `location /` + `try_files $uri $uri/ =404` | Coba file persis sesuai URI, lalu sebagai direktori, jika tidak ada kembalikan 404. |

**Debug**

```bash
nginx -t -c /root/myconfig/nginx.conf      # uji sintaks tanpa menjalankan
```

Output sukses: `syntax is ok` dan `test is successful`. Jika error, pesan menyebut nomor baris. Kesalahan umum: lupa `;`, kurung kurawal tidak seimbang, lupa blok `events`.

---

#### Langkah A.1.6 - Buat `start_http.sh`

```bash
#!/bin/bash
nginx -c /root/myconfig/nginx.conf -g 'daemon off;'
```

**Teori**

- `-c` memakai file config kita, bukan default `/etc/nginx/nginx.conf`.
- `-g 'daemon off;'` membuat nginx berjalan di **foreground**. Ini penting untuk node GNS3/Docker: proses yang menjadi *start command* harus tetap hidup, kalau nginx masuk background (daemonize) proses utama langsung selesai dan container dianggap berhenti.
- Shebang `#!/bin/bash`: pada image Alpine mungkin hanya ada `sh`. Jika `bash: not found`, ganti ke `#!/bin/sh` atau `apk add bash`.

**Debug:** `cat -A /root/myconfig/start_http.sh | head -3` (pastikan tidak ada karakter `^M` hasil copy dari Windows).

---

#### Langkah A.1.7 - Beri izin eksekusi

```bash
cd /root/myconfig
chmod +x start_http.sh
```

**Teori:** Linux menolak mengeksekusi file tanpa bit *execute* (`Permission denied`). Perintah `chmod +x` harus dijalankan **di direktori tempat file berada** (atau dengan path lengkap).

**Debug:** `ls -l /root/myconfig/start_http.sh` → harus ada `x` (`-rwxr-xr-x`).

---

#### Langkah A.1.8 - Jalankan web server

```bash
cd /root/myconfig
./start_http.sh
```

**Teori:** Dengan `daemon off`, terminal akan "tertahan" (menampilkan log) selama nginx jalan. Untuk menjalankannya sambil tetap memantau, pakai `tmux` (satu pane nginx, pane lain untuk debugging).

**Debug (di Kazdel, di terminal/pane lain)**

```bash
netstat -tlnp | grep :80          # nginx listen di port 80?  (atau: ss -tlnp)
ps | grep nginx                   # proses master + worker ada?
curl -I http://127.0.0.1          # tes lokal, harus HTTP/1.1 200 OK
tail -f /root/mylogs/error.log    # lihat error realtime
```

| Gejala | Penyebab / solusi |
|---|---|
| `bind() to 0.0.0.0:80 failed (98: Address in use)` | nginx lain sudah jalan: `pkill nginx` lalu start ulang |
| `open() "/root/mylogs/error.log" failed` | Direktori `/root/mylogs` belum dibuat |
| `unknown directive` | Typo di nginx.conf (`nginx -t`) |
| `403 Forbidden` | Permission file/direktori, atau `index.html` tidak ada |
| `404 Not Found` | Salah `root`, atau nama file bukan `index.html` |
| `curl` lokal OK tapi dari Sargon gagal | Masalah jaringan (ping dulu), bukan nginx |

---

### Indikator Keberhasilan A.1

**1. Verifikasi lewat browser (Ognisko)**

1. Buka console Ognisko via VNC viewer.
2. Klik kanan desktop → *Desktops → chromium* (pada tangkapan layar tampil Firefox; pakai browser yang tersedia).
3. Buka `http://[IP Prefix].200.11`.
4. Harus tampil tulisan **"This is my web"**.

**2. Verifikasi lewat command line (Sargon)**

```bash
curl -vL http://[IP Prefix].200.11
```

Contoh hasil yang diharapkan:

```
* Trying [IP Prefix].200.11:80...
* Connected to [IP Prefix].200.11 ([IP Prefix].200.11) port 80
> GET / HTTP/1.1
> Host: [IP Prefix].200.11
> User-Agent: curl/8.14.1
> Accept: */*
< HTTP/1.1 200 OK
< Server: nginx/1.26.3
< Content-Type: text/html
< Content-Length: 111
< Connection: keep-alive
< ETag: "68d7964b-6f"
<html> ... This is my web ... </html>
* Connection #0 to host [IP Prefix].200.11 left intact
```

**Teori: cara membaca output `curl -v`**

| Baris | Arti |
|---|---|
| `* Trying ...:80` / `* Connected` | TCP 3-way handshake ke port 80 berhasil (lapisan Transport) |
| `> GET / HTTP/1.1`, `> Host:` | **Request** dikirim client; `Host` header menentukan *virtual host* mana yang dipilih server |
| `< HTTP/1.1 200 OK` | **Status code** respons; 200 = sukses |
| `< Server: nginx` | Software server |
| `< Content-Length`, `ETag`, `Last-Modified` | Metadata konten untuk caching/validasi |
| `-L` | Ikuti redirect (3xx) otomatis |

**Debug lintas node**

```bash
# Dari Sargon
ping -c 2 [IP Prefix].200.11          # L3 OK?
curl -v --connect-timeout 3 http://[IP Prefix].200.11
```

| Hasil curl | Artinya |
|---|---|
| `Connection refused` | Host terjangkau tapi tidak ada proses listen di port itu (nginx belum jalan / salah port) |
| `Connection timed out` | Paket tidak sampai/dibuang (firewall, routing, atau IP salah) |
| `Could not resolve host` | Masalah DNS (Bagian B) |
| `HTTP 404/403` | nginx jalan, masalah di konfigurasi/konten |

---

## B. Bagian B - DNS Server

### Tujuan Pembelajaran

1. Memahami DNS sebagai mekanisme pemetaan nama domain → IP.
2. Memasang DNS server (BIND) dan mengintegrasikannya dengan DHCP agar semua client otomatis mengetahui alamat DNS server.

### Tools

- GNS3 appliance `netics-pc` dan `netics-pc-desktop`
- BIND (`named`)
- DHCP server yang sudah aktif

### Teori singkat DNS

DNS adalah "buku telepon internet": manusia mengingat nama (`www.netics.my.id`), mesin berkomunikasi dengan IP. Komponen penting:

- **Zone:** potongan namespace yang dikelola satu server (mis. `netics.my.id`).
- **Authoritative / master server:** server yang memegang data asli suatu zone (kasus Chernobog).
- **Resource Record (RR):** entri di dalam zone: `A` (nama → IPv4), `NS` (siapa name server zone), `SOA` (metadata zone).
- **Resolver:** komponen di client yang menanyakan DNS (dikonfigurasi lewat `/etc/resolv.conf`).

### Problem B.1 - Setup DNS Server

**Soal:** Pasang DNS server pada Chernobog (`[IP Prefix].200.12/24`) untuk domain `netics.my.id`, dan integrasikan dengan DHCP agar semua client otomatis mengetahui DNS server ini.

---

#### Langkah B.1.1 - Set IP Chernobog

Seperti Langkah A.1.1, dengan `address [IP Prefix].200.12`, `netmask 255.255.255.0`, `gateway [IP Prefix].200.1`.

**Teori:** DNS server wajib statis. IP-nya dituliskan di opsi DHCP dan di record `ns1`; jika berubah semua client kehilangan DNS.

**Debug:** `ip addr show eth0`, `ip route`, `ping -c 2 [IP Prefix].200.1`.

#### Langkah B.1.2 - Buat struktur direktori

```bash
mkdir -p /root/dns
mkdir -p /root/dnsdata
```

**Teori:** `/root/dns` menyimpan config, zone file, dan log (`directory "/root/dns"` di named.conf). Path relatif pada `file "db.xxx"` dihitung dari direktori ini.

**Debug:** `ls -ld /root/dns /root/dnsdata`.

#### Langkah B.1.3 - Install BIND

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apk update
apk add bind
```

Opsional untuk debugging (menyediakan `dig`, `nslookup`):

```bash
apk add bind-tools
```

**Teori:** BIND (`named`) adalah implementasi DNS server paling umum. `bind-tools` terpisah dari server karena berisi tool *client* (dig/host/nslookup/named-checkzone).

**Debug:** `which named`, `named -v`.

#### Langkah B.1.4 - Buat `named.conf` di `/root/dns`

```
options {
    directory "/root/dns";
    listen-on { any; };
    allow-query { any; };
};

logging {
    channel default_log {
        file "/root/dns/named.log" versions 3 size 5m;
        severity info;
        print-time yes;
        print-severity yes;
        print-category yes;
    };
    category default { default_log; };
    category queries { default_log; };
};

zone "localhost" {
    type master;
    file "db.localhost";
};

zone "netics.my.id" {
    type master;
    file "db.netics.my.id";
};
```

**Teori**

| Bagian | Fungsi |
|---|---|
| `directory` | Direktori kerja named; semua `file "..."` relatif ke sini |
| `listen-on { any; }` | named menerima query di semua interface IPv4 (default hanya localhost, sehingga client tidak bisa bertanya) |
| `allow-query { any; }` | Siapa saja boleh bertanya (di produksi dibatasi) |
| `logging` | Mencatat aktivitas ke `named.log` (rotasi 3 versi × 5 MB). Kategori `queries` mencatat **setiap query**, sangat berguna untuk memverifikasi bahwa client benar-benar bertanya ke server ini |
| `zone ... type master` | Server ini **otoritatif utama** untuk zone tersebut; data diambil dari file zone |

**Debug**

```bash
named-checkconf /root/dns/named.conf        # sintaks named.conf (tidak ada output = OK)
```

#### Langkah B.1.5 - Buat `db.localhost`

```
$TTL 86400
@   IN  SOA localhost. root.localhost. (
        1       ; serial
        3600    ; refresh
        1800    ; retry
        604800  ; expire
        86400 ) ; minimum
    IN  NS  localhost.
    IN  A   127.0.0.1
```

**Teori:** Zone `localhost` memetakan nama `localhost` ke `127.0.0.1` (loopback). Zone ini standar agar resolver lokal tidak error.

#### Langkah B.1.6 - Buat `db.netics.my.id`

```
$TTL 86400
@   IN  SOA ns1.netics.my.id. admin.netics.my.id. (
        1       ; serial
        3600    ; refresh
        1800    ; retry
        604800  ; expire
        86400 ) ; minimum
    IN  NS  ns1.netics.my.id.
ns1 IN  A   [IP Prefix].200.12
www IN  A   [IP Prefix].200.11
```

**Teori: anatomi zone file**

| Elemen | Penjelasan |
|---|---|
| `$TTL 86400` | Default *Time To Live* (24 jam): berapa lama resolver boleh menyimpan jawaban di cache |
| `@` | Singkatan nama zone (`netics.my.id.`) |
| `SOA` | *Start of Authority*: wajib ada di tiap zone. `ns1.netics.my.id.` = primary NS; `admin.netics.my.id.` = email admin (titik pertama = `@`) |
| `serial` | Nomor versi zone; **naikkan setiap kali mengedit** agar slave/cache tahu ada perubahan |
| `refresh/retry/expire/minimum` | Parameter sinkronisasi slave dan *negative caching* |
| `IN NS ns1...` | Menyatakan `ns1` adalah name server zone ini |
| `ns1 IN A ...` | Nama `ns1.netics.my.id` → IP Chernobog (*glue*; NS harus punya record A) |
| `www IN A ...` | `www.netics.my.id` → IP web server Kazdel (inti tujuan kita) |
| **Titik di akhir (`.`)** | Menandakan *fully qualified*. Tanpa titik, BIND menambahkan nama zone di belakangnya (`ns1.netics.my.id` → `ns1.netics.my.id.netics.my.id.`). Ini kesalahan paling umum. |

**Debug**

```bash
named-checkzone localhost /root/dns/db.localhost
named-checkzone netics.my.id /root/dns/db.netics.my.id    # harus: "OK"
```

#### Langkah B.1.7 - Buat `start_dns.sh` dan jalankan

```bash
#!/bin/sh
named -g -c /root/dns/named.conf
```

```bash
chmod +x /root/dns/start_dns.sh
/root/dns/start_dns.sh
```

Pantau aktivitas (di pane tmux lain):

```bash
tail -f /root/dns/named.log
```

**Teori:** `-g` = *foreground* (log juga ke layar) - alasan sama dengan `daemon off` di nginx. `-c` menunjuk file config. Pada log pertama akan muncul banyak pesan `network unreachable resolving './NS/IN'` (lihat Gambar B.1 pada soal): ini **wajar**. named mencoba *priming* ke root server internet dan gagal karena Chernobog tidak punya akses internet/IPv6; tidak mempengaruhi zone lokal yang otoritatif.

**Debug**

```bash
ps | grep named
netstat -ulnp | grep :53         # UDP 53 listen?
netstat -tlnp | grep :53         # TCP 53 listen?
dig @127.0.0.1 www.netics.my.id  # tes lokal di Chernobog (apk add bind-tools)
dig @[IP Prefix].200.12 www.netics.my.id +short     # harus [IP Prefix].200.11
```

| Gejala | Penyebab |
|---|---|
| `named` langsung exit | Error config; baca output foreground atau `named-checkconf` |
| `could not open file ... named.log` | Direktori/izin salah |
| `dig` → `SERVFAIL` | Zone file bermasalah (`named-checkzone`) |
| `dig` → `NXDOMAIN` | Nama tidak ada di zone (typo, titik hilang) |
| `dig` → `connection timed out` | named tidak listen / firewall / `listen-on` salah |
| Perubahan zone tidak terbaca | Serial belum dinaikkan, named belum di-reload (`rndc reload` atau restart node) |

#### Langkah B.1.8 - Pantau console Chernobog

Klik kanan node → **Console** untuk melihat proses `named`. Gunakan `tmux` agar bisa menjalankan `named` di satu pane dan `tail -f named.log` di pane lain.

```bash
apk add tmux
tmux                      # Ctrl+b lalu %  = bagi pane vertikal; Ctrl+b lalu panah = pindah pane
```

---

### Integrasi dengan DHCP

**Tujuan:** Semua client otomatis tahu alamat DNS server tanpa diset satu per satu.

#### Langkah B.1.9 - Edit `dhcpd.conf` di Aegir

Soal: tambahkan `option dns [IP Prefix].200.12 8.8.8.8`.

Padanan sintaks ISC dhcpd:

```
option domain-name-servers [IP Prefix].200.12, 8.8.8.8;
option domain-name "netics.my.id";
```

**Teori**

- `domain-name-servers` memberi daftar DNS yang ditulis client sebagai `nameserver` di `/etc/resolv.conf`. Urutan penting: client bertanya ke **yang pertama** (`.200.12`); `8.8.8.8` jadi cadangan.
- `domain-name` membuat baris `search netics.my.id` (seperti pada Gambar B.3), sehingga client bisa mengetik `ping www` dan otomatis dilengkapi menjadi `www.netics.my.id`.
- **Peringatan:** karena `8.8.8.8` sekunder, jika Chernobog salah menjawab (NXDOMAIN) client **tidak** otomatis beralih ke 8.8.8.8. Fallback hanya terjadi bila server pertama tidak merespons.

**Debug**

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf       # cek sintaks sebelum restart
```

#### Langkah B.1.10 - Reload DHCP, restart seluruh client

Restart/reload dhcpd di Aegir, lalu restart client (atau jalankan ulang `udhcpc -i eth0`).

**Teori:** DNS dibagikan lewat **DHCP lease**, bukan push. Client baru mengetahui perubahan pada saat meminta lease baru.

**Debug (di client)**

```bash
udhcpc -i eth0           # minta ulang lease
cat /etc/resolv.conf     # harus ada nameserver [IP Prefix].200.12
```

---

### Indikator Keberhasilan B.1

**1. Di Sargon:**

```bash
cat /etc/resolv.conf
```

Contoh output yang benar:

```
search netics.my.id
nameserver [IP Prefix].200.12
```

**2. Ping dan curl memakai nama domain:**

```bash
ping www.netics.my.id
curl -vL http://www.netics.my.id
```

**3. Dari Ognisko**, buka `http://www.netics.my.id` lewat browser.
**4. Ulangi** dari netics-pc lain untuk memastikan konsistensi.

**Teori:** `ping` hanya membuktikan resolusi nama → IP (IP yang dilaporkan harus `[IP Prefix].200.11`). `curl -v` menunjukkan baris `Host www.netics.my.id was resolved` lalu header `Host: www.netics.my.id`, membuktikan seluruh rantai DNS → TCP → HTTP bekerja.

**Debug (bertingkat, dari client)**

```bash
cat /etc/resolv.conf                           # 1. client tahu DNS server?
ping -c 2 [IP Prefix].200.12                   # 2. DNS server terjangkau?
dig @[IP Prefix].200.12 www.netics.my.id       # 3. DNS server menjawab langsung?
dig www.netics.my.id                           # 4. client memakai resolver default?
nslookup www.netics.my.id                      # alternatif jika dig tak ada
tail -f /root/dns/named.log                    # (di Chernobog) apakah query client masuk?
```

| Gejala | Diagnosis |
|---|---|
| `resolv.conf` kosong / tidak ada `.200.12` | DHCP belum diperbarui / `option domain-name-servers` salah |
| `dig @IP` berhasil, tapi `ping nama` gagal | `resolv.conf` tidak menunjuk ke Chernobog |
| `ping nama` → `bad address` | Resolver tidak menemukan jawaban (cek NXDOMAIN, titik FQDN) |
| Query tidak muncul di `named.log` | Client bertanya ke server lain (mis. 8.8.8.8) atau paket tidak sampai |
| IP hasil resolve salah | Record `www` di zone file salah atau cache lama (tunggu TTL / restart named) |

---

### Problem B.2 - Multi-Zone DNS dan Multi-Domain Web Server

**Soal:** Perluas DNS server agar melayani dua zona (`netics.my.id` dan `netics.org`), dan perluas web server agar melayani nama domain berbeda dengan lokasi index berbeda: `http://www.netics.my.id/` dan `http://web.netics.org`.

#### Bagian DNS Server (Chernobog)

**Langkah B.2.1 - Tambahkan zona di `named.conf`**

```
zone "netics.org" {
    type master;
    file "db.netics.org";
};
```

**Teori:** Satu instance BIND dapat menjadi otoritatif untuk banyak zone sekaligus; tiap zone cukup satu blok `zone` dan satu file data.

**Langkah B.2.2 - Buat `db.netics.org`**

```
$TTL 86400
@   IN  SOA ns1.netics.org. admin.netics.org. (
        1       ; serial
        3600    ; refresh
        1800    ; retry
        604800  ; expire
        86400 ) ; minimum
    IN  NS  ns1.netics.org.
ns1 IN  A   [IP Prefix].200.12
web IN  A   [IP Prefix].200.11
```

**Teori:** Kedua domain (`www.netics.my.id` dan `web.netics.org`) menunjuk ke **IP yang sama** (`.200.11`). Pembeda keduanya baru terjadi di lapisan HTTP (header `Host`), bukan di DNS.

**Langkah B.2.3 - Restart Chernobog** (stop lalu start) agar konfigurasi baru terbaca.

**Debug**

```bash
named-checkconf /root/dns/named.conf
named-checkzone netics.org /root/dns/db.netics.org
dig @[IP Prefix].200.12 web.netics.org +short        # [IP Prefix].200.11
dig @[IP Prefix].200.12 ns1.netics.org +short        # [IP Prefix].200.12
```

**Indikator:** dari Sargon dan client lain, `ping ns1.netics.org` harus resolve ke `[IP Prefix].200.12`.

#### Bagian Web Server (Kazdel)

**Langkah B.2.4 - Buat direktori konten baru**

```bash
mkdir -p /root/myweb-netics-org
```

**Langkah B.2.5 - Buat `index.html` di direktori tersebut**

```html
<html>
<head>
<title>Hello</title>
</head>
<body>
<p>
This is NETICS-ORG web
</p>
</body>
</html>
```

**Langkah B.2.6 - Ubah blok `http` pada `nginx.conf`**

```nginx
http {
    server {
        listen 80;
        server_name www.netics.my.id;

        root /root/myweb;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }

    server {
        listen 80;
        server_name web.netics.org;

        root /root/myweb-netics-org;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

**Teori: name-based virtual host**

Satu IP dan satu port dapat melayani banyak situs. nginx membaca header `Host:` pada request lalu mencocokkannya dengan `server_name`:

```
Request  Host: www.netics.my.id  →  server block 1  →  /root/myweb
Request  Host: web.netics.org    →  server block 2  →  /root/myweb-netics-org
Host tidak cocok keduanya        →  server block pertama (default)
```

Itu sebabnya `curl http://[IP Prefix].200.11` (tanpa nama) akan memberi konten `www.netics.my.id` (server pertama = default).

**Langkah B.2.7 - Restart Kazdel.**

**Debug**

```bash
nginx -t -c /root/myconfig/nginx.conf

# Uji virtual host tanpa DNS, dengan memalsukan header Host:
curl -H "Host: www.netics.my.id" http://[IP Prefix].200.11
curl -H "Host: web.netics.org"   http://[IP Prefix].200.11

# Atau paksa resolve:
curl --resolve web.netics.org:80:[IP Prefix].200.11 http://web.netics.org
```

Jika kedua perintah itu benar tapi curl memakai nama gagal → masalah DNS. Jika keduanya mengembalikan konten yang sama → `server_name` salah/typo atau nginx belum di-restart.

### Indikator Keberhasilan B.2

1. Dari Sargon & netics-pc lain: `ping web.netics.org` dan `ping www.netics.my.id` sama-sama resolve ke `[IP Prefix].200.11`.
2. Dari Iberia:

   ```bash
   curl -vL http://www.netics.my.id     # "This is my web"
   curl -vL http://web.netics.org       # "This is NETICS-ORG web"
   ```

   Konten, `Content-Length`, dan `ETag` kedua respons harus **berbeda** (file berbeda).
3. Dari Ognisko, buka kedua alamat lewat browser.

---

## C. Bagian C - Reverse Proxy

### Tujuan Pembelajaran

1. Memahami pemisahan fungsi antara **reverse proxy** dan **real server**.
2. Mengonfigurasi reverse proxy yang meneruskan request berdasarkan **path URL** ke backend berbeda, termasuk backend di port berbeda.

### Tools

- GNS3 appliance `netics-pc` dan `netics-pc-desktop`
- nginx (reverse proxy dan real server)
- Python3 (untuk backend aplikasi dinamis)

### Teori: Reverse Proxy

```
 Internet users ──► www.netics.my.id ──► [Reverse Proxy] ──┬─► Nginx  (web1 :8080)
                                                           ├─► Nginx  (web2 :8080)
                                                           └─► Python (web2 :5000)
```

| | Forward proxy | Reverse proxy |
|---|---|---|
| Mewakili | **Client** | **Server** |
| Letak | Dekat client | Di depan server |
| Tujuan | Menyembunyikan/menyaring client | Menyembunyikan backend, satu pintu masuk |

Manfaat reverse proxy: **satu domain dan satu port** untuk banyak layanan, backend tidak terekspos ke luar, mudah menambah TLS/caching/load balancing, dan layanan bisa dipindah tanpa mengubah URL yang dilihat user.

### Problem C.1 - Reverse Proxy di Depan Dua Web Server

**Soal:** Implementasikan reverse proxy yang melayani base URL `http://www.netics.my.id`, meneruskan `/web1` ke Web1 (`http://[IP Prefix].200.20:8080`) dan `/web2` ke Web2 (`http://[IP Prefix].200.21:8080`).

#### Langkah Kerja, Web1 (web1)

**Langkah C.1.1 - Set IP `[IP Prefix].200.20/24`**

```
# Static config for eth0
auto eth0
iface eth0 inet static
        address [IP Prefix].200.20
        netmask 255.255.255.0
        gateway [IP Prefix].200.1
```

**Teori & debug:** sama seperti A.1.1. Perhatikan pada tangkapan layar Web2 baris `gateway` ter-comment (`#`): **uncomment** agar ada default route. `ip addr`, `ip route`, `ping [IP Prefix].200.1`.

**Langkah C.1.2 - Buat struktur direktori** (`myconfig`, `myweb`, `mylogs`) seperti Bagian A.

**Langkah C.1.3 - Install nginx** (perintah sama dengan A.1.2).

**Langkah C.1.4 - Buat `index.html` berisi "Ini WEB-1"**

```html
<html>
<head>
<title>This is My Web</title>
</head>
<body>
<p>
Ini WEB-1
</p>
</body>
</html>
```

**Teori:** Teks yang berbeda di tiap backend adalah "sidik jari": dengan melihat teks di hasil akhir kita tahu persis backend mana yang menjawab.

**Langkah C.1.5 - Buat `nginx.conf` (listen 8080)**

```nginx
user root;
worker_processes auto;
worker_cpu_affinity auto;
pid /tmp/nginx.pid;
error_log /root/mylogs/error.log;

events { worker_connections 768; }

http {
    server {
        listen 8080;
        server_name _;

        root /root/myweb;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

**Teori:** Port 80 akan dipakai reverse proxy (pintu publik); backend memakai **8080** agar (a) tidak bentrok bila server dan proxy berada di mesin yang sama, dan (b) jelas membedakan "port publik" dan "port internal". Perhatikan: **tidak ada** `server` duplikat/`proxy` di backend, ia hanya web server biasa.

**Langkah C.1.6 - Buat `start_http.sh`** (sama dengan Bagian A) dan set *start command* node (GNS3: *Configure → Advanced / Start command*) agar nginx otomatis jalan saat node dinyalakan.

**Langkah C.1.7 - Restart Web1, verifikasi dari Sargon dan Ognisko**

```bash
curl http://[IP Prefix].200.20:8080
```

Output harus memuat **"Ini WEB-1"**.

**Debug**

```bash
# Di web1
netstat -tlnp | grep 8080
curl http://127.0.0.1:8080
nginx -t -c /root/myconfig/nginx.conf
tail /root/mylogs/error.log
# Dari client
ping -c 2 [IP Prefix].200.20
curl -v --connect-timeout 3 http://[IP Prefix].200.20:8080
```

| Gejala | Penyebab |
|---|---|
| `Connection refused` pada :8080 | nginx tidak jalan / masih `listen 80` |
| Timeout | IP/gateway salah atau kabel ke Switch3 |
| Muncul halaman default nginx | `-c` tidak menunjuk file config kita |

#### Langkah Kerja, Web2 (web2)

Ulangi persis Web1 dengan perbedaan:

- IP `[IP Prefix].200.21/24`
- Isi `index.html`: **"Ini WEB-2"** (soal menulis "WEB2-2" di satu tempat dan "WEB-2" di tempat lain; konsistenlah satu versi)
- `nginx.conf` dan `start_http.sh` identik dengan Web1

```html
<html>
<head>
<title>This is My Web</title>
</head>
<body>
<p>
Ini WEB-2
</p>
</body>
</html>
```

Verifikasi:

```bash
curl http://[IP Prefix].200.21:8080
```

Debug sama dengan Web1 (ganti IP).

#### Langkah Kerja, Reverse Proxy (Kazdel)

**Langkah C.1.8 - Pastikan IP Kazdel tetap `[IP Prefix].200.11/24`** dan `server_name` tetap `www.netics.my.id`.

**Langkah C.1.9 - Ubah `nginx.conf` menjadi reverse proxy**

Berikut file lengkap (blok `events` dan header global tetap diperlukan):

```nginx
user root;
worker_processes auto;
worker_cpu_affinity auto;
pid /tmp/nginx.pid;
error_log /root/mylogs/error.log;

events { worker_connections 768; }

http {
    server {
        listen 80;
        server_name www.netics.my.id;

        location = /web1 {
            proxy_pass http://[IP Prefix].200.20:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location = /web2 {
            proxy_pass http://[IP Prefix].200.21:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

> **Peringatan (Multi-domain):** konfigurasi di atas menggantikan blok `http` Bagian A/B. Verifikasi Menyeluruh nomor 5 meminta `web.netics.org` tetap berfungsi, jadi **pertahankan** `server { server_name web.netics.org; root /root/myweb-netics-org; ... }` dari Problem B.2 di dalam blok `http` yang sama.

**Teori: arti tiap direktif**

| Direktif | Penjelasan |
|---|---|
| `location = /web1` | Cocok **persis** `/web1` (tanda `=` = exact match). `/web1/` atau `/web1/x` **tidak** cocok (jadi 404). Alternatif prefix: `location /web1/` + `proxy_pass http://.../;` memetakan seluruh sub-path. |
| `proxy_pass http://IP:8080/;` | Meneruskan request ke backend. Karena `proxy_pass` memakai **URI** (`/` di akhir), bagian path yang cocok digantikan: `/web1` → `/` di backend. Tanpa `/` di akhir, backend akan menerima `/web1` yang mungkin tidak ada (404). |
| `proxy_set_header Host $host` | Meneruskan header Host asli client (`www.netics.my.id`). Default nginx mengganti Host dengan alamat upstream (`IP:8080`), sehingga backend yang memakai virtual host akan salah memilih situs. |
| `proxy_set_header X-Real-IP $remote_addr` | Karena backend hanya melihat IP **proxy** sebagai sumber koneksi, IP client asli dititipkan lewat header ini agar bisa dicatat/dipakai backend. |

Alur request `curl http://www.netics.my.id/web1` dari Sargon:

1. Sargon bertanya DNS ke Chernobog → `[IP Prefix].200.11`.
2. Sargon membuka TCP ke Kazdel:80, `GET /web1`, `Host: www.netics.my.id`.
3. nginx Kazdel mencocokkan `location = /web1` → membuka koneksi **baru** ke `[IP Prefix].200.20:8080`, mengirim `GET /` dengan `Host` dan `X-Real-IP`.
4. Web1 menjawab "Ini WEB-1" → Kazdel meneruskan ke Sargon. Sargon tidak pernah tahu Web1 ada.

**Langkah C.1.10 - Restart node Kazdel.**

**Debug**

```bash
nginx -t -c /root/myconfig/nginx.conf
tail -f /root/mylogs/error.log
# Di Kazdel, tes jalur ke backend (bukan lewat proxy):
curl http://[IP Prefix].200.20:8080
curl http://[IP Prefix].200.21:8080
# Tes proxy:
curl -v http://www.netics.my.id/web1
curl -v http://www.netics.my.id/web2
```

| Gejala | Penyebab |
|---|---|
| `502 Bad Gateway` | Proxy tak bisa terhubung ke backend (backend mati, IP/port salah, Kazdel tidak punya rute ke subnet backend). Cek `error.log`: `connect() failed (111: Connection refused)` / `(113: No route to host)` |
| `504 Gateway Timeout` | Backend tidak merespons dalam batas waktu |
| `404` dari nginx proxy | Path tidak cocok `location` (mis. `/web1/` dengan `location = /web1`) |
| `404` dari backend | `proxy_pass` tanpa `/` di akhir sehingga backend menerima `/web1` |
| Konten salah | `proxy_pass` salah IP, atau `server_name` tidak cocok sehingga masuk server block lain |
| `nginx: [emerg] "proxy_pass" cannot have URI part in location given by regular expression` | `proxy_pass` ber-URI dipakai di dalam `location ~` (regex) |

### Indikator Keberhasilan C.1

**1. Dari Sargon:**

```bash
curl http://www.netics.my.id/web1      # Ini WEB-1
curl http://www.netics.my.id/web2      # Ini WEB-2
```

Dua backend berbeda diakses dari satu domain dan satu port (80).

**2. Dari Ognisko**, cek kedua path lewat browser.

---

### Problem C.2 - Menambahkan Backend Aplikasi di Port Berbeda

**Soal:** Aplikasi kalender berbasis Python berada di Web2, port 5000. Perluas reverse proxy agar `/app` diarahkan ke `http://[IP Prefix].200.21:5000`, selain `/web1` dan `/web2`.

#### Langkah Kerja, Aplikasi di Web2

**Langkah C.2.1 - Buat direktori**

```bash
mkdir -p /root/aplikasi
```

**Langkah C.2.2 - Install Python**

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apk update
apk add python3
```

**Teori:** Aplikasi memakai library standar Python (`http.server`, `calendar`, `datetime`), sehingga **tidak perlu** pip/Flask meskipun caption soal menyebut Flask.

**Debug:** `python3 --version`.

**Langkah C.2.3 - Buat `/root/aplikasi/program.py`**

```python
import calendar
from datetime import datetime
from http.server import BaseHTTPRequestHandler, HTTPServer

PORT = 5000


class CalendarRequestHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        # 1. Get current year and month
        now = datetime.now()
        year = now.year
        month = now.month

        # 2. Generate calendar data
        cal = calendar.Calendar(firstweekday=6)  # 6 = Sunday start. Change to 0 for Monday.
        month_days = cal.monthdayscalendar(year, month)
        month_name = calendar.month_name[month]

        # 3. Build HTML response
        html = f"""<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{month_name} {year} - Calendar</title>
    <style>
        body {{
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background-color: #f4f7f6;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }}
        .calendar-container {{
            background: #ffffff;
            padding: 24px;
            border-radius: 12px;
            box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
            width: 100%;
            max-width: 500px;
        }}
        h2 {{
            text-align: center;
            color: #2c3e50;
            margin-top: 0;
            margin-bottom: 20px;
        }}
        table {{
            width: 100%;
            border-collapse: collapse;
        }}
        th {{
            background-color: #3498db;
            color: white;
            font-weight: 600;
            padding: 12px 0;
            width: 14.28%;
            border-radius: 4px;
        }}
        td {{
            text-align: center;
            padding: 16px 0;
            color: #333;
            font-size: 16px;
            font-weight: 500;
        }}
        .today {{
            background-color: #e8f4fd;
            color: #3498db;
            border-radius: 50%;
            font-weight: bold;
        }}
        .empty {{
            color: #ccc;
        }}
    </style>
</head>
<body>
    <div class="calendar-container">
        <h2>{month_name} {year}</h2>
        <table>
            <thead>
                <tr>
                    <th>Sun</th>
                    <th>Mon</th>
                    <th>Tue</th>
                    <th>Wed</th>
                    <th>Thu</th>
                    <th>Fri</th>
                    <th>Sat</th>
                </tr>
            </thead>
            <tbody>
"""
        # 4. Construct table rows from calendar data
        for week in month_days:
            html += "<tr>"
            for day in week:
                if day == 0:
                    html += '<td class="empty">&bull;</td>'
                elif day == now.day:
                    html += f'<td class="today">{day}</td>'
                else:
                    html += f"<td>{day}</td>"
            html += "</tr>"

        html += """
            </tbody>
        </table>
    </div>
</body>
</html>
"""
        # 5. Send HTTP Response headers and content
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(html.encode("utf-8"))


def run():
    server_address = ("", PORT)
    httpd = HTTPServer(server_address, CalendarRequestHandler)
    print(f"Server running at http://localhost:{PORT}/")
    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        print("\nServer stopped.")


if __name__ == "__main__":
    run()
```

**Teori**

- `HTTPServer(("", 5000), Handler)`: alamat `""` berarti **semua interface** (0.0.0.0). Jika diganti `127.0.0.1`, aplikasi hanya bisa diakses dari mesin itu sendiri, dan proxy di Kazdel tidak bisa menjangkaunya.
- `do_GET` dipanggil untuk setiap request GET; handler membuat HTML kalender bulan berjalan secara dinamis (berbeda dengan `index.html` yang statis).
- `{{ }}` pada f-string = tanda kurung kurawal literal CSS.
- Kalender menampilkan **tanggal sistem node** (bukan "tahun 2025" seperti tertulis di soal): jika jam node salah, tampilan akan salah.

**Debug**

```bash
python3 -m py_compile /root/aplikasi/program.py    # cek sintaks / indentasi
date                                               # tanggal sistem node
```

**Langkah C.2.4 - Jalankan aplikasi**

```bash
python3 /root/aplikasi/program.py
```

**Teori:** Proses ini berjalan di foreground dan menahan terminal. Jalankan di pane `tmux` terpisah, atau di background (`python3 ... &`) / daftarkan pada *start command* node agar otomatis jalan setiap boot. Setiap request tercatat di terminal (mis. `"GET / HTTP/1.1" 200`).

**Debug**

```bash
netstat -tlnp | grep 5000           # harus listen di 0.0.0.0:5000
curl http://127.0.0.1:5000          # tes lokal
```

| Gejala | Penyebab |
|---|---|
| `Address already in use` | Instance lama masih hidup: `pkill -f program.py` |
| `SyntaxError` / `IndentationError` | Indentasi rusak saat copy-paste |
| `python3: not found` | `apk add python3` belum berhasil |

**Langkah C.2.5 - Verifikasi awal dari Ognisko/client**

```bash
curl http://[IP Prefix].200.21:5000
```

Harus muncul HTML kalender (browser: tabel kalender bulan berjalan).

**Teori:** Verifikasi **langsung ke backend dulu** sebelum lewat proxy. Dengan begitu bila nanti `/app` gagal, kita tahu masalah ada di konfigurasi proxy, bukan di aplikasinya (*divide and conquer*).

#### Langkah Kerja, Reverse Proxy (Kazdel)

**Langkah C.2.6 - Pastikan IP Kazdel tidak berubah** (`ip addr show eth0`).

**Langkah C.2.7 - Tambahkan satu blok `location` baru**

```nginx
http {
    server {
        listen 80;
        server_name www.netics.my.id;

        location = /web1 {
            proxy_pass http://[IP Prefix].200.20:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location = /web2 {
            proxy_pass http://[IP Prefix].200.21:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location = /app {
            proxy_pass http://[IP Prefix].200.21:5000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

**Teori:** Port tujuan hanyalah bagian dari `proxy_pass`. Proxy bisa meneruskan ke **port berbeda pada host yang sama** (`:8080` untuk nginx, `:5000` untuk Python) sehingga dari sisi user, ketiga layanan tampak berada di `www.netics.my.id` port 80.

**Langkah C.2.8 - Restart Kazdel.**

**Debug**

```bash
nginx -t -c /root/myconfig/nginx.conf
curl -v http://www.netics.my.id/app
curl http://[IP Prefix].200.21:5000          # dari Kazdel: backend terjangkau?
tail -f /root/mylogs/error.log
```

| Gejala | Penyebab |
|---|---|
| `502` pada `/app` | Program Python tidak berjalan / bind ke 127.0.0.1 / port salah |
| `/web1` & `/web2` OK, `/app` 404 | Blok `location = /app` belum ada atau nginx belum di-restart |
| `/app` OK tapi terkena `/app/` 404 | Karena exact-match; akses tepat `/app` |

### Indikator Keberhasilan C.2

**Dari Sargon:**

```bash
curl http://www.netics.my.id/web1
curl http://www.netics.my.id/web2
curl http://www.netics.my.id/app      # HTML kalender (bulan/tahun berjalan)
```

Dari Ognisko, cek ketiga path lewat browser.

---

## E. Verifikasi Menyeluruh

Setelah Problem C.1 dan C.2 selesai, uji DNS, HTTP server, dan reverse proxy sebagai satu kesatuan.

| No. | Item Verifikasi | Perintah / Cara | Dari Node |
|---|---|---|---|
| 1 | Client otomatis memiliki alamat DNS server | `cat /etc/resolv.conf` | Sargon |
| 2 | Resolusi nama domain berhasil | `ping www.netics.my.id`, `ping web.netics.org` | Sargon, netics-pc lain |
| 3 | Akses langsung ke tiap backend (tanpa proxy) masih berfungsi | `curl http://[IP Prefix].200.20:8080`, `curl http://[IP Prefix].200.21:8080`, `curl http://[IP Prefix].200.21:5000` | Sargon |
| 4 | Reverse proxy meneruskan ke backend yang benar | `curl http://www.netics.my.id/web1`, `/web2`, `/app` | Sargon, Iberia |
| 5 | Multi-domain web server tetap berfungsi | `curl http://www.netics.my.id`, `curl http://web.netics.org` | Sargon |
| 6 | Verifikasi visual lewat browser | Browser ke seluruh alamat pada baris 2-5 | Ognisko |
| 7 | Header diteruskan dengan benar (Host, X-Real-IP) | `curl -vL` ke tiap path, amati header request/response | Sargon |

**Teori & debug tiap baris**

| No. | Mengapa diuji | Jika gagal, periksa |
|---|---|---|
| 1 | Membuktikan integrasi DHCP→DNS | Opsi `domain-name-servers`, lease diperbarui |
| 2 | Membuktikan DNS multi-zone | `dig @[IP Prefix].200.12 ...`, `named.log`, serial/zona |
| 3 | Memisahkan masalah backend vs proxy (backend harus tetap hidup meski proxy mati) | Proses nginx/python di web1/web2, `netstat -tlnp` |
| 4 | Inti reverse proxy | `error.log` Kazdel, `proxy_pass`, exact-match |
| 5 | Memastikan virtual host tidak hilang setelah proxy dikonfigurasi | Blok `server_name web.netics.org` masih ada di nginx.conf Kazdel? |
| 6 | Mensimulasikan pengguna sebenarnya (browser) | Cache browser (coba *hard refresh* / jendela privat), DNS dari Ognisko |
| 7 | Menjamin backend mengetahui host dan IP client asli. `X-Real-IP` hanya terlihat pada sisi **proxy → backend** (lihat Tugas Mandiri no. 4) | `proxy_set_header` ada di tiap location |

> Pada sisi **client → proxy** (`curl -v`), header request menampilkan `Host: www.netics.my.id`. Header `X-Real-IP` ditambahkan oleh proxy sehingga tidak muncul pada output curl client; ia hanya dapat dibuktikan pada log backend atau capture paket di sisi proxy → backend.

---

## F. Tugas Mandiri Bagian C

**Topologi tambahan:** `web3` terhubung ke Switch3 (bersama web1 dan web2), IP `[IP Prefix].200.22/24`, gateway `[IP Prefix].200.1`, web server port 8080.

### Soal

1. Konfigurasi mengikuti bagian sebelumnya (Bagian A hingga E). **Tambahan:** tunjukkan juga topologi yang kamu buat di GNS3.
2. Tambahkan Web3 (backend HTTP baru) dan integrasikan ke reverse proxy lewat path `/web3`.
3. Daftarkan nama domain baru di DNS server (misalnya `app.netics.my.id`) yang mengarah ke reverse proxy, lalu ubah konfigurasi reverse proxy agar juga bisa diakses lewat nama domain baru ini (name-based virtual host), selain `www.netics.my.id`.
4. Buktikan dengan `curl` dan capture (`termshark`) pada Kazdel (gunakan `tmux` agar nginx dan termshark berjalan bersamaan) bahwa `X-Real-IP` pada request ke Web3 sesuai dengan IP address client yang sesungguhnya mengakses.

### Panduan Pengerjaan, dengan Teori dan Debug

#### Soal 1 - Topologi GNS3

Ambil screenshot topologi (Router+NAT, Switch1, Switch2 beserta 4 client, Aegir, Kazdel, Chernobog, Switch3 beserta web1/web2/web3) lengkap dengan label IP seperti pada Gambar C.9/f. **Teori:** dokumentasi topologi dipakai untuk memverifikasi bahwa segmen L2 benar (semua di broadcast domain `[IP Prefix].200.0/24`).

#### Soal 2 - Web3 dan `/web3`

**Langkah**

1. Di web3: IP `[IP Prefix].200.22/24` + gateway; install nginx; struktur direktori; `index.html` berisi **"Ini WEB-3"**; `nginx.conf` identik web1 (listen 8080); `start_http.sh` + start command.
2. Di Kazdel, tambahkan:

```nginx
location = /web3 {
    proxy_pass http://[IP Prefix].200.22:8080/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

3. Restart Kazdel.

**Teori:** Menambah backend baru hanya perlu satu `location` baru: inilah skalabilitas reverse proxy: URL publik sama, jumlah backend bertambah tanpa user menyadarinya.

**Debug**

```bash
curl http://[IP Prefix].200.22:8080         # langsung ke backend
curl http://www.netics.my.id/web3           # lewat proxy -> "Ini WEB-3"
```

`502` → backend web3 belum jalan / IP salah. `404` → location belum ditambahkan / nginx belum restart.

#### Soal 3 - Domain baru `app.netics.my.id` (name-based virtual host)

**a) DNS (Chernobog)** - tambahkan ke `db.netics.my.id` dan **naikkan serial**:

```
$TTL 86400
@   IN  SOA ns1.netics.my.id. admin.netics.my.id. (
        2       ; serial (dinaikkan dari 1)
        3600
        1800
        604800
        86400 )
    IN  NS  ns1.netics.my.id.
ns1 IN  A   [IP Prefix].200.12
www IN  A   [IP Prefix].200.11
app IN  A   [IP Prefix].200.11
```

`app` menunjuk ke **IP reverse proxy (Kazdel)**, bukan ke backend. Restart named / node Chernobog.

**b) Reverse proxy (Kazdel)** - cara paling sederhana, satu blok server melayani dua nama:

```nginx
server {
    listen 80;
    server_name www.netics.my.id app.netics.my.id;
    ...  # semua location /web1 /web2 /web3 /app
}
```

Atau blok `server` terpisah untuk `app.netics.my.id` bila ingin perilaku berbeda (mis. `location /` langsung ke Python di `:5000`).

**Teori:** DNS memetakan **dua nama ke satu IP**; nginx membedakan keduanya dari header `Host`. `$host` yang diteruskan ke backend akan berisi nama yang dipakai client.

**Debug**

```bash
dig @[IP Prefix].200.12 app.netics.my.id +short       # [IP Prefix].200.11
curl -H "Host: app.netics.my.id" http://[IP Prefix].200.11/web1   # tes tanpa DNS
curl http://app.netics.my.id/web3
```

Jika `dig` menjawab NXDOMAIN → serial/restart/typo (titik) pada zone. Jika konten selalu dari server block pertama → `server_name` belum menyertakan nama baru.

#### Soal 4 - Bukti `X-Real-IP` (curl + termshark)

**a) Siapkan tools di Kazdel**

```bash
apk add tmux termshark       # (jika termshark tidak ada di repo: apk add tshark, atau pakai tcpdump)
tmux
```

- Pane 1: `cd /root/myconfig && ./start_http.sh` (nginx).
- Pane 2 (Ctrl+b lalu `%`): `termshark -i eth0 -f "tcp port 8080 and host [IP Prefix].200.22"`.

**b) Dari client (mis. Sargon)**

```bash
ip addr show eth0                          # catat IP client, mis. [IP Prefix].200.101
curl -v http://www.netics.my.id/web3
```

**c) Amati pada termshark**

Cari paket HTTP `GET / HTTP/1.1` dari Kazdel (`.200.11`) ke web3 (`.200.22:8080`) dan buka detail *Hypertext Transfer Protocol*. Harus terlihat:

```
Host: www.netics.my.id
X-Real-IP: [IP Prefix].200.101      <-- sama dengan IP client yang sebenarnya
```

Sedangkan *source IP* paket itu adalah Kazdel (`.200.11`): **inilah alasan** header `X-Real-IP` dibutuhkan (backend tidak bisa mengetahui IP client dari koneksi TCP).

**d) Bukti tambahan di web3** (opsional tapi kuat): catat header di log backend dengan `log_format`:

```nginx
http {
    log_format realip '$remote_addr real=$http_x_real_ip "$request" $status';
    access_log /root/mylogs/access.log realip;
    ...
}
```

```bash
tail -f /root/mylogs/access.log
# [IP Prefix].200.11 real=[IP Prefix].200.101 "GET / HTTP/1.1" 200
```

**Teori:** `$remote_addr` pada web3 = IP proxy; `$http_x_real_ip` = isi header `X-Real-IP` yang dititipkan proxy = IP client.

**Debug**

| Gejala | Penyebab |
|---|---|
| termshark tidak menangkap paket | Interface salah (`-i eth0`), atau filter terlalu ketat; coba tanpa `-f` |
| Hanya melihat IP Kazdel, tak ada `X-Real-IP` | `proxy_set_header X-Real-IP $remote_addr;` belum ada di `location /web3` |
| `X-Real-IP` menunjukkan IP Kazdel | Ada proxy lain di depan, atau `curl` dijalankan dari Kazdel sendiri |
| Tidak bisa melihat isi header | Trafik terenkripsi (HTTPS); di lab ini pakai HTTP polos sehingga terbaca |
| termshark tampilan hang/berantakan | Perlu terminal ukuran cukup besar; pakai `tmux` dan perbesar pane |

---

## Catatan Koreksi terhadap Soal Asli

Beberapa ketidakkonsistenan pada dokumen sumber yang perlu diperhatikan saat praktik:

| # | Temuan | Rekomendasi |
|---|---|---|
| 1 | Soal menulis `option router` dan `option dns` | Nama baku ISC dhcpd adalah `option routers` dan `option domain-name-servers`. Ikuti Modul 1 bila ia mendefinisikan alias |
| 2 | Caption menyebut "Flask", padahal kode memakai `http.server` bawaan Python | Tidak perlu `pip install flask` |
| 3 | Teks "Ini WEB2-2" vs "Ini WEB-2" | Pakailah satu versi secara konsisten (disarankan "Ini WEB-2") |
| 4 | Teks "kalender tahun 2025" pada indikator `/app` | Kalender menampilkan tanggal sistem node (screenshot menunjukkan Oktober 2026) |
| 5 | Teks menyebut Chromium, screenshot menampilkan Firefox | Pakai browser yang tersedia di Ognisko |
| 6 | `chmod +x start_http.sh` ditulis sebelum berpindah ke `/root/myconfig` | Jalankan `cd /root/myconfig` lebih dulu |
| 7 | Konfigurasi proxy Problem C.1 hanya memuat satu `server` (`www.netics.my.id`) | Pertahankan blok `web.netics.org` agar Verifikasi Menyeluruh no. 5 lulus |
| 8 | `location = /web1` (exact match) | `/web1/` memberi 404. Opsional: gunakan `location /web1/` dengan `proxy_pass http://.../;` |
| 9 | Gateway pada interface web2 ter-comment pada screenshot | Uncomment agar node punya default route |
| 10 | Kode Python mengandung f-string `{{ }}` | Jangan diubah; itu kurung kurawal literal untuk CSS |

---

## Cheat Sheet Debugging

### Metode umum: periksa dari lapisan bawah ke atas

| Lapisan | Pertanyaan | Perintah |
|---|---|---|
| L1/L2 | Interface UP? Kabel tersambung ke switch yang benar? | `ip link`, lihat warna link di GNS3 |
| L3 | IP/gateway benar? Rute ada? | `ip addr`, `ip route`, `ping <gw>` |
| L3 (jauh) | Bisa mencapai internet/host lain? | `ping 8.8.8.8`, `traceroute <ip>` |
| DNS | Nama ter-resolve? | `cat /etc/resolv.conf`, `dig @<dns> <nama>`, `nslookup` |
| L4 | Port terbuka? | `netstat -tlnp`, `curl -v`, `nc -zv <ip> <port>` |
| L7 | Aplikasi menjawab benar? | `curl -v`, `tail /root/mylogs/error.log`, `/root/dns/named.log` |

### Tabel kode status HTTP

| Kode | Arti | Penyebab umum di lab ini |
|---|---|---|
| 200 | OK | - |
| 301/302 | Redirect | `-L` mengikuti |
| 403 | Forbidden | Permission file / tanpa index |
| 404 | Not Found | Path salah, `location` tidak cocok, `proxy_pass` tanpa `/` |
| 502 | Bad Gateway | Backend mati / salah IP:port |
| 504 | Gateway Timeout | Backend tak merespons / rute putus |

### Perintah cepat

```bash
# Konfigurasi
nginx -t -c /root/myconfig/nginx.conf
named-checkconf /root/dns/named.conf
named-checkzone <zone> <file>
dhcpd -t -cf /etc/dhcp/dhcpd.conf

# Proses & port
ps | grep -E "nginx|named|python"
netstat -tlnp          # TCP listen
netstat -ulnp          # UDP listen (DNS)
pkill nginx; pkill named

# Uji end-to-end
dig @[IP Prefix].200.12 www.netics.my.id +short
curl -v http://www.netics.my.id/web1
curl -H "Host: web.netics.org" http://[IP Prefix].200.11
curl --resolve www.netics.my.id:80:[IP Prefix].200.11 http://www.netics.my.id/

# Penangkapan paket
tcpdump -ni eth0 port 53 or port 80 or port 8080
termshark -i eth0 -f "tcp port 8080"
```

### Daftar periksa sebelum menyerahkan laporan

- [ ] Router: NAT aktif, `ip_forward=1`, ping google.com berhasil
- [ ] Aegir: DHCP jalan, pool `.100-.150`, `routers` dan `domain-name-servers` terkirim
- [ ] Kazdel: IP statis `.11`, nginx jalan, dua virtual host + proxy `/web1 /web2 /web3 /app`
- [ ] Chernobog: IP statis `.12`, named jalan, zone `netics.my.id` dan `netics.org` (+ `app`)
- [ ] web1/web2/web3: IP statis, nginx :8080; web2 + Python :5000; semua punya *start command*
- [ ] Semua verifikasi tabel E lulus (screenshot curl dan browser)
- [ ] Bukti `X-Real-IP` (termshark + log)
