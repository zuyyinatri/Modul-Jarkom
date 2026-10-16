# Pra-Praktikum Modul 2 - Soal Kedua
## Integrasi DNS, Reverse Proxy, dan Tiga Backend (Studi Kasus Sekolah)

> **Topologi:** sama dengan soal sebelumnya (Router+NAT, Switch1/2/3, Sargon, Iberia, Higashi, Ognisko, Aegir, Kazdel, Chernobog, web1, web2, web3).
> `[IP Prefix]` diganti sesuai pembagian prefix. Ganti sebelum menyalin perintah.
> Soal ini adalah **integrasi** dari Bagian 0, A, B, dan C pada soal pertama. Detail dasar tiap komponen (netmask, interfaces, dll.) mengikuti dokumen `Pra-Praktikum_Modul_2.md`.

---

## Daftar Isi

1. [Soal](#soal)
2. [Poin Keberhasilan](#poin-keberhasilan)
3. [Pemetaan Node dan Rencana Alamat](#pemetaan-node-dan-rencana-alamat)
4. [Langkah 1 - Konektivitas Internet (Router, Gateway, DHCP)](#langkah-1---konektivitas-internet-router-gateway-dhcp)
5. [Langkah 2 - DNS Server Internal + Integrasi DHCP](#langkah-2---dns-server-internal--integrasi-dhcp)
6. [Langkah 3 - Tiga Backend (web1, web2, web3)](#langkah-3---tiga-backend-web1-web2-web3)
7. [Langkah 4 - Reverse Proxy di Kazdel](#langkah-4---reverse-proxy-di-kazdel)
8. [Langkah 5 - Pembuktian (curl dan Browser)](#langkah-5---pembuktian-curl-dan-browser)
9. [Langkah 6 - Jawaban Dua Pertanyaan Konsep](#langkah-6---jawaban-dua-pertanyaan-konsep)
10. [Cheat Sheet Debugging](#cheat-sheet-debugging)
11. [Checklist Laporan](#checklist-laporan)

---

## Soal

1. Koneksikan seluruh topologi agar bisa mendapat internet dari NAT (konfigurasi router, gateway, dan DHCP Server).
2. Pasang DNS server internal dengan domain pilihan sendiri (misalnya `sekolah.local`) yang mengarah ke Kazdel, dan integrasikan dengan DHCP server agar seluruh PC lab (DHCP client) otomatis mengetahui alamat DNS ini.
3. Jalankan tiga backend HTTP/aplikasi berbeda (**nilai**, **absensi**, **kalender akademik**) di tiga node berbeda (**web1, web2, dan web3**), menggunakan nginx:
   - a. `web1.html`
   - b. `web2.html`
   - c. `web3.html`
4. Pasang satu reverse proxy yang bisa diakses lewat nama domain (misalnya `www.sekolah.local`), meneruskan `/nilai`, `/absensi`, dan `/kalender` ke masing-masing backend.
5. Buktikan dari command line (curl) dan browser bahwa ketiga aplikasi dapat diakses lewat satu nama domain yang sama.
6. Jelaskan pada asisten 2 pertanyaan berikut:
   - a. Jelaskan bagaimana DNS, reverse proxy, dan backend saling bekerja sama sebagai satu sistem.
   - b. Serta apa yang terjadi bila salah satu komponen tidak berfungsi.

## Poin Keberhasilan

- [ ] Seluruh client di jaringan otomatis memiliki setting DNS yang benar (dibuktikan lewat `/etc/resolv.conf`).
- [ ] Ketiga aplikasi dapat diakses lewat **satu nama domain dan port yang sama**, dibuktikan dengan curl dan browser.
- [ ] Backend **tidak pernah diakses langsung** oleh client dalam skenario akhir ini; seluruh traffic melalui reverse proxy.
- [ ] Menjelaskan kerja sama DNS, reverse proxy, dan backend sebagai satu sistem.

---

## Pemetaan Node dan Rencana Alamat

| Node | Peran pada soal ini | IP | Catatan |
|---|---|---|---|
| Router | Gateway + NAT ke internet | eth0 DHCP (NAT), eth1 `[IP Prefix].200.1/24` | MASQUERADE + ip_forward |
| Aegir | DHCP Server | `[IP Prefix].200.10/24` (statis) | Pool `.100-.150`, bagikan gateway + DNS |
| Chernobog | DNS Server internal | `[IP Prefix].200.12/24` (statis) | Zone `sekolah.local` |
| Kazdel | **Reverse proxy** | `[IP Prefix].200.11/24` (statis) | Target domain `www.sekolah.local` |
| web1 | Backend **Nilai** | `[IP Prefix].200.20/24`, port 8080 | `web1.html` |
| web2 | Backend **Absensi** | `[IP Prefix].200.21/24`, port 8080 | `web2.html` |
| web3 | Backend **Kalender Akademik** | `[IP Prefix].200.22/24`, port 8080 | `web3.html` |
| Sargon, Iberia, Higashi | Client CLI (PC lab) | DHCP | Uji `curl` |
| Ognisko | Client browser (PC lab) | DHCP | Uji browser via VNC |

**Pemetaan URL (hasil akhir):**

| URL publik | Diteruskan oleh Kazdel ke | File di backend |
|---|---|---|
| `http://www.sekolah.local/nilai` | `http://[IP Prefix].200.20:8080/web1.html` | web1.html |
| `http://www.sekolah.local/absensi` | `http://[IP Prefix].200.21:8080/web2.html` | web2.html |
| `http://www.sekolah.local/kalender` | `http://[IP Prefix].200.22:8080/web3.html` | web3.html |

**Teori: mengapa Kazdel menjadi tujuan DNS?** Soal meminta DNS "mengarah ke Kazdel". Artinya semua nama (mis. `www.sekolah.local`) di-resolve ke **IP reverse proxy**, bukan ke backend. Client hanya mengenal satu alamat (Kazdel); alamat backend (`.20/.21/.22`) disembunyikan. Ini yang memenuhi poin "backend tidak pernah diakses langsung".

> **Catatan tentang `.local`:** akhiran `.local` dicadangkan untuk **mDNS** (RFC 6762). Sebagian resolver/browser (terutama yang memakai `systemd-resolved`, Avahi, atau Chromium di beberapa OS) mencoba mDNS lebih dulu dan bisa tidak menanyakan DNS server Anda. Pada Alpine (musl) umumnya aman, tetapi bila `.local` gagal di browser Ognisko sementara `dig` berhasil, ganti domain menjadi mis. `sekolah.lan` atau `sekolah.test` (soal memperbolehkan domain pilihan sendiri). Debug-nya ada di Langkah 2.

---

## Langkah 1 - Konektivitas Internet (Router, Gateway, DHCP)

**Tujuan:** semua node bisa keluar ke internet melalui Router (NAT). Dibutuhkan agar `apk update`/`apk add` bekerja dan sebagai bukti topologi terkoneksi.

### 1.1 Router: interface, forwarding, NAT

Interfaces Router:

```
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
        address [IP Prefix].200.1
        netmask 255.255.255.0
```

NAT dan forwarding:

```bash
apk update
apk add iptables
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -j MASQUERADE -o eth0 -s [IP Prefix].0.0/16
```

**Teori**

| Komponen | Mengapa dibutuhkan |
|---|---|
| eth1 `.200.1` statis | IP ini dipakai semua node sebagai **default gateway**; harus tetap |
| `ip_forward=1` | Tanpa ini kernel tidak meneruskan paket antar-interface sehingga Router tidak berfungsi sebagai router |
| `MASQUERADE` | IP lab privat tidak bisa dirutekan di internet; paket ditulis ulang agar tampak berasal dari IP eth0 Router, dan balasan dikembalikan lewat tabel *conntrack* |

**Debug**

```bash
ip addr show eth0; ip addr show eth1
ip route                                   # default via <gateway NAT> dev eth0
sysctl net.ipv4.ip_forward                 # = 1
iptables -t nat -L POSTROUTING -n -v       # rule ada, counter naik saat ada traffic
iptables -L FORWARD -n -v                  # policy DROP? -> tambahkan ACCEPT
ping -c 3 8.8.8.8 && ping -c 3 google.com
tcpdump -ni eth0 icmp                      # source IP harus sudah berubah ke IP eth0
```

### 1.2 Aegir: IP statis, DNS sementara, dan DHCP server

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

`dhcpd.conf` (versi awal; opsi DNS ditambahkan pada Langkah 2):

```
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet [IP Prefix].200.0 netmask 255.255.255.0 {
    range [IP Prefix].200.100 [IP Prefix].200.150;
    option routers [IP Prefix].200.1;
}
```

> Nama baku ISC dhcpd adalah `option routers` (soal modul menulis `option router`; ikuti Modul 1 bila alias tersebut memang didefinisikan).

**Teori:** DHCP membagikan IP + parameter jaringan (gateway) otomatis. `option routers` = *default gateway* client; tanpa itu client hanya bisa berkomunikasi di subnet yang sama, tidak bisa ke internet. Server (`.10-.22`) berada di luar pool agar tak bentrok dengan lease.

**Debug**

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf          # cek sintaks
dhcpd -4 -f -d eth0                        # foreground + log DORA
cat /var/lib/dhcp/dhcpd.leases
# Di client:
udhcpc -i eth0 ; ip addr ; ip route        # default via [IP Prefix].200.1 ?
```

### 1.3 Semua server statis: gateway ke Router

Untuk Kazdel, Chernobog, web1, web2, web3 isi interfaces statis dengan `gateway [IP Prefix].200.1` (pastikan baris `gateway` tidak ter-comment).

**Debug**

```bash
ping -c 2 [IP Prefix].200.1
ping -c 2 8.8.8.8
```

### Bukti Langkah 1

- Router: `ping google.com` berhasil.
- Tiap node: `ping 8.8.8.8` berhasil.
- Client: mendapat IP `.100-.150` dan default route `.200.1`.

---

## Langkah 2 - DNS Server Internal + Integrasi DHCP

**Tujuan:** domain internal `sekolah.local` yang mengarah ke Kazdel, dan seluruh PC lab otomatis memakai DNS ini.

### 2.1 Chernobog: struktur dan instalasi BIND

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apk update
apk add bind bind-tools tmux
mkdir -p /root/dns /root/dnsdata
```

**Teori:** `bind-tools` menyediakan `dig`, `nslookup`, `named-checkzone`; `tmux` memungkinkan menjalankan `named` dan memantau log berdampingan.

### 2.2 `/root/dns/named.conf`

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

zone "sekolah.local" {
    type master;
    file "db.sekolah.local";
};
```

*(Jika Anda masih ingin mempertahankan zona `netics.my.id`/`netics.org` dari soal pertama, biarkan blok zone-nya tetap ada; BIND dapat melayani banyak zone.)*

**Teori:** `listen-on { any; }` agar client di jaringan bisa bertanya (default hanya localhost). `type master` = Chernobog adalah sumber data otoritatif untuk `sekolah.local`. `queries` dicatat agar kita bisa **membuktikan** client benar-benar bertanya ke DNS ini.

### 2.3 `db.localhost`

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

### 2.4 `db.sekolah.local`

```
$TTL 86400
@   IN  SOA ns1.sekolah.local. admin.sekolah.local. (
        1       ; serial
        3600    ; refresh
        1800    ; retry
        604800  ; expire
        86400 ) ; minimum
    IN  NS  ns1.sekolah.local.
ns1 IN  A   [IP Prefix].200.12
www IN  A   [IP Prefix].200.11
```

**Teori**

| Record | Fungsi |
|---|---|
| `SOA` | Metadata wajib zone (primary NS, email admin, serial, timer) |
| `NS` + `ns1 A` | Menyatakan siapa name server zone dan IP-nya (*glue*) |
| `www A [IP Prefix].200.11` | **Inti soal**: nama `www.sekolah.local` mengarah ke **Kazdel (reverse proxy)** |
| Titik di akhir FQDN | Tanpa titik, BIND menambahkan nama zone sehingga namanya salah ganda |
| `serial` | Naikkan setiap mengedit zone |

> Jika ingin nama tambahan (mis. `sekolah.local` polos mengarah ke proxy) tambahkan `@ IN A [IP Prefix].200.11`.

**Debug sebelum menjalankan**

```bash
named-checkconf /root/dns/named.conf
named-checkzone sekolah.local /root/dns/db.sekolah.local    # harus "OK"
```

### 2.5 `start_dns.sh` dan jalankan

```bash
cat > /root/dns/start_dns.sh <<'EOF'
#!/bin/sh
named -g -c /root/dns/named.conf
EOF
chmod +x /root/dns/start_dns.sh
/root/dns/start_dns.sh
```

Di pane tmux lain: `tail -f /root/dns/named.log`.

**Teori:** `-g` = foreground (wajib agar proses tetap hidup sebagai start command node). Pesan `network unreachable resolving './NS/IN'` pada log awal **normal** (named mencoba menghubungi root server internet, tidak memengaruhi zone lokal).

**Debug**

```bash
ps | grep named
netstat -ulnp | grep :53 ; netstat -tlnp | grep :53
dig @127.0.0.1 www.sekolah.local +short                # [IP Prefix].200.11
dig @[IP Prefix].200.12 www.sekolah.local +short       # dari node lain
```

| Gejala | Penyebab |
|---|---|
| `named` langsung keluar | Error config/zone (baca output foreground) |
| `SERVFAIL` | Zone file salah (`named-checkzone`) |
| `NXDOMAIN` | Nama tidak ada di zone (typo, titik FQDN hilang) |
| `timed out` | named tidak listen / jaringan / `listen-on` |
| Perubahan tidak terbaca | Serial belum naik, named belum di-restart |

### 2.6 Integrasi DHCP (Aegir)

Tambahkan di subnet pada `dhcpd.conf`:

```
option domain-name-servers [IP Prefix].200.12, 8.8.8.8;
option domain-name "sekolah.local";
```

(Padanan dari `option dns [IP Prefix].200.12 8.8.8.8` pada modul.)

Reload/restart DHCP server, lalu semua client meminta lease ulang (restart node atau `udhcpc -i eth0`).

**Teori**

- `domain-name-servers` → baris `nameserver` di `/etc/resolv.conf` client; yang pertama (`.12`) ditanya lebih dulu, `8.8.8.8` hanya cadangan jika server pertama tidak merespons.
- `domain-name` → baris `search sekolah.local`, sehingga mengetik `ping www` otomatis menjadi `www.sekolah.local`.
- DNS dibagikan **saat lease**, bukan di-*push*; client lama belum tahu sampai memperbarui lease.

**Debug**

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
# Di client:
udhcpc -i eth0
cat /etc/resolv.conf
dig www.sekolah.local +short
```

### Bukti Langkah 2 (poin keberhasilan #1)

Di Sargon (dan client lain):

```bash
cat /etc/resolv.conf
```

Harus tampil kira-kira:

```
search sekolah.local
nameserver [IP Prefix].200.12
```

Lalu:

```bash
ping -c 3 www.sekolah.local        # resolve ke [IP Prefix].200.11
```

| Gejala | Penyebab |
|---|---|
| `resolv.conf` tidak berisi `.200.12` | DHCP belum diperbarui / `domain-name-servers` salah |
| `dig @.12` berhasil, `ping nama` gagal | `resolv.conf` client tidak menunjuk ke Chernobog |
| `dig` OK tetapi **browser** gagal untuk `.local` | Perilaku mDNS pada `.local`; ganti ke `sekolah.lan` atau `sekolah.test` |
| Query tak muncul di `named.log` | Client bertanya ke server lain / paket tidak sampai |

---

## Langkah 3 - Tiga Backend (web1, web2, web3)

**Tujuan:** tiga aplikasi berbeda, masing-masing di node sendiri, dilayani nginx. Backend hanya melayani isi; yang menghadap publik nanti adalah Kazdel.

| Node | IP | Aplikasi | File |
|---|---|---|---|
| web1 | `[IP Prefix].200.20` | Nilai | `web1.html` |
| web2 | `[IP Prefix].200.21` | Absensi | `web2.html` |
| web3 | `[IP Prefix].200.22` | Kalender akademik | `web3.html` |

> File `web1.html`, `web2.html`, `web3.html` adalah berkas yang disediakan pada soal (tautan). Letakkan masing-masing di `/root/myweb/` pada node yang bersangkutan. Jika belum punya, gunakan contoh minimal pada langkah 3.4 untuk pengujian.

Ulangi langkah berikut pada **setiap** backend (ganti IP dan file).

### 3.1 IP statis

```
auto eth0
iface eth0 inet static
        address [IP Prefix].200.20       # .21 untuk web2, .22 untuk web3
        netmask 255.255.255.0
        gateway [IP Prefix].200.1
```

**Debug:** `ip addr show eth0`, `ip route`, `ping -c 2 [IP Prefix].200.1`.

### 3.2 Install nginx dan siapkan direktori

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
apk update
apk add nginx
mkdir -p /root/myconfig /root/myweb /root/mylogs
```

**Teori:** Konvensi direktori `myconfig/myweb/mylogs` sama dengan soal pertama; log terpusat mempermudah debugging.

### 3.3 `nginx.conf` backend (port 8080)

```nginx
user root;
worker_processes auto;
worker_cpu_affinity auto;
pid /tmp/nginx.pid;
error_log /root/mylogs/error.log;

events { worker_connections 768; }

http {
    access_log /root/mylogs/access.log;

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

**Teori**

- Port **8080** = port internal (bukan 80) sehingga jelas dibedakan dari pintu publik (proxy di port 80).
- `access_log` diaktifkan agar kita bisa membuktikan **siapa** yang mengakses backend (nanti harus selalu IP Kazdel, bukan client) - ini bukti poin "backend tidak diakses langsung".
- `try_files` menyajikan file apa adanya: `/web1.html` → `/root/myweb/web1.html`.

### 3.4 Konten (contoh minimal bila file asli belum ada)

web1 (`/root/myweb/web1.html`):

```html
<html><head><title>Nilai</title></head>
<body><h1>Aplikasi Nilai</h1><p>Ini WEB-1 (backend nilai)</p></body></html>
```

web2 (`/root/myweb/web2.html`):

```html
<html><head><title>Absensi</title></head>
<body><h1>Aplikasi Absensi</h1><p>Ini WEB-2 (backend absensi)</p></body></html>
```

web3 (`/root/myweb/web3.html`):

```html
<html><head><title>Kalender Akademik</title></head>
<body><h1>Kalender Akademik</h1><p>Ini WEB-3 (backend kalender)</p></body></html>
```

**Teori:** Teks berbeda di tiap backend berfungsi sebagai "sidik jari" sehingga dari hasil akhir terlihat backend mana yang menjawab.

### 3.5 `start_http.sh` dan start command

```bash
cat > /root/myconfig/start_http.sh <<'EOF'
#!/bin/sh
nginx -c /root/myconfig/nginx.conf -g 'daemon off;'
EOF
cd /root/myconfig && chmod +x start_http.sh && ./start_http.sh
```

Set juga *start command* node di GNS3 agar nginx otomatis jalan saat node dinyalakan.

**Teori:** `daemon off` membuat nginx tetap di foreground; wajib untuk proses utama container/node.

### 3.6 Uji langsung ke backend (tahap pengembangan saja)

```bash
# dari Kazdel
curl http://[IP Prefix].200.20:8080/web1.html
curl http://[IP Prefix].200.21:8080/web2.html
curl http://[IP Prefix].200.22:8080/web3.html
```

**Teori:** Tes langsung ke backend **sebelum** memasang proxy memisahkan masalah (*divide and conquer*): jika nanti proxy gagal, kita tahu backend sehat.

**Debug (per backend)**

```bash
nginx -t -c /root/myconfig/nginx.conf
netstat -tlnp | grep 8080
curl -I http://127.0.0.1:8080/web1.html
tail -f /root/mylogs/error.log /root/mylogs/access.log
```

| Gejala | Penyebab |
|---|---|
| `Connection refused` | nginx belum jalan / masih listen 80 |
| `Timeout` | IP/gateway salah, kabel ke Switch3 |
| `404` | File tidak ada di `/root/myweb` atau salah nama |
| `403` | Permission atau `user` nginx tidak bisa membaca `/root` |

### 3.7 (Opsional, memperkuat poin "backend tidak diakses langsung")

Batasi backend agar hanya menerima koneksi dari Kazdel. Tambahkan di blok `server` backend:

```nginx
        allow [IP Prefix].200.11;   # Kazdel
        deny  all;
```

Dengan ini akses langsung dari Sargon/Ognisko ke `:8080` mendapat `403 Forbidden`, sedangkan lewat proxy tetap `200`. Lakukan pengujian backend langsung (3.6) **dari Kazdel**.

---

## Langkah 4 - Reverse Proxy di Kazdel

**Tujuan:** satu pintu masuk `www.sekolah.local` port 80, meneruskan tiga path ke tiga backend.

### 4.1 `/root/myconfig/nginx.conf` (Kazdel)

```nginx
user root;
worker_processes auto;
worker_cpu_affinity auto;
pid /tmp/nginx.pid;
error_log /root/mylogs/error.log;

events { worker_connections 768; }

http {
    access_log /root/mylogs/access.log;

    server {
        listen 80;
        server_name www.sekolah.local;

        location = /nilai {
            proxy_pass http://[IP Prefix].200.20:8080/web1.html;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location = /absensi {
            proxy_pass http://[IP Prefix].200.21:8080/web2.html;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location = /kalender {
            proxy_pass http://[IP Prefix].200.22:8080/web3.html;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

> Jika Kazdel sebelumnya melayani `www.netics.my.id` / `web.netics.org`, blok `server` tersebut boleh dipertahankan di dalam `http { }` yang sama; nginx memilih blok berdasarkan header `Host`.

**Teori**

| Direktif | Penjelasan |
|---|---|
| `server_name www.sekolah.local` | Blok ini hanya aktif untuk request dengan header `Host: www.sekolah.local` (name-based virtual host) |
| `location = /nilai` | Cocok **persis** `/nilai`. `/nilai/` atau `/nilai/x` tidak cocok (404) |
| `proxy_pass http://IP:8080/web1.html` | Karena `proxy_pass` memakai URI, bagian path yang cocok (`/nilai`) **diganti** menjadi `/web1.html` pada request ke backend |
| `Host $host` | Meneruskan nama host asli ke backend (default nginx menggantinya dengan `IP:port` upstream) |
| `X-Real-IP $remote_addr` | Menitipkan IP client asli karena backend hanya melihat IP Kazdel sebagai sumber koneksi |

**Alternatif bila halaman backend memuat aset relatif (CSS/JS/gambar):** gunakan prefix-match + rewrite, mis.

```nginx
location /nilai/ {
    proxy_pass http://[IP Prefix].200.20:8080/;
    ...
}
location = /nilai { return 301 /nilai/; }
```

lalu simpan halaman utama backend sebagai `index.html`. Dengan `location = /nilai` + `proxy_pass .../web1.html`, tautan relatif di dalam halaman tidak akan ter-resolve.

### 4.2 Jalankan / restart

```bash
nginx -t -c /root/myconfig/nginx.conf
cd /root/myconfig && ./start_http.sh        # atau restart node Kazdel
```

**Debug**

```bash
nginx -t -c /root/myconfig/nginx.conf
netstat -tlnp | grep :80
tail -f /root/mylogs/error.log
curl -v -H "Host: www.sekolah.local" http://127.0.0.1/nilai     # uji tanpa DNS, dari Kazdel
```

| Gejala | Penyebab |
|---|---|
| `502 Bad Gateway` | Backend mati / IP:port salah / Kazdel tak punya rute. `error.log`: `connect() failed (111/113)` |
| `504 Gateway Timeout` | Backend tidak merespons |
| `404` dari proxy | Path tidak cocok `location` (mis. `/nilai/` dengan exact match) |
| `404` dari backend | Nama file pada `proxy_pass` salah (`web1.html`) |
| `403` dari backend | `allow/deny` (langkah 3.7) tidak menyertakan IP Kazdel |
| Konten dari server block lain | `server_name` tidak cocok dengan header `Host` yang dikirim |
| Perubahan tidak berlaku | nginx belum di-restart / masih proses lama (`pkill nginx`) |

---

## Langkah 5 - Pembuktian (curl dan Browser)

### 5.1 Dari command line (Sargon, Iberia, Higashi)

```bash
cat /etc/resolv.conf                              # bukti DNS dari DHCP
curl -v http://www.sekolah.local/nilai            # -> backend nilai (web1)
curl -v http://www.sekolah.local/absensi          # -> backend absensi (web2)
curl -v http://www.sekolah.local/kalender         # -> backend kalender (web3)
```

**Cara membaca output `curl -v`**

| Baris | Arti |
|---|---|
| `* Host www.sekolah.local:80 was resolved` | DNS berhasil (IP yang dilaporkan harus `[IP Prefix].200.11`) |
| `* Connected to ... port 80` | TCP ke **proxy** (bukan backend) |
| `> GET /nilai HTTP/1.1`, `> Host: www.sekolah.local` | Request yang dikirim client |
| `< HTTP/1.1 200 OK`, `< Server: nginx` | Respons sukses dari proxy |
| Isi body "Aplikasi Nilai" | Terbukti berasal dari web1 |

Satu domain, satu port (80), tiga aplikasi berbeda.

### 5.2 Dari browser (Ognisko via VNC)

Buka tiga URL:

- `http://www.sekolah.local/nilai`
- `http://www.sekolah.local/absensi`
- `http://www.sekolah.local/kalender`

Ambil screenshot ketiganya (address bar harus terlihat).

### 5.3 Bukti "backend tidak pernah diakses langsung oleh client"

Di tiap backend:

```bash
tail /root/mylogs/access.log
```

Setiap baris harus berasal dari **`[IP Prefix].200.11` (Kazdel)**, tidak ada IP client (`.100-.150`). Bila memakai pembatasan 3.7, uji juga:

```bash
# dari Sargon
curl -I http://[IP Prefix].200.20:8080/web1.html      # 403 Forbidden
```

Opsional, capture di Kazdel untuk melihat dua sisi koneksi:

```bash
tcpdump -ni eth0 'port 80 or port 8080'
```

Terlihat: client → Kazdel:80, lalu Kazdel → backend:8080 (dua koneksi TCP terpisah).

**Debug bertingkat bila pembuktian gagal**

```bash
# 1. DNS
dig www.sekolah.local +short                  # [IP Prefix].200.11 ?
# 2. Proxy terjangkau
ping -c 2 [IP Prefix].200.11
curl -v --connect-timeout 3 http://[IP Prefix].200.11/nilai -H "Host: www.sekolah.local"
# 3. Backend (dari Kazdel)
curl http://[IP Prefix].200.20:8080/web1.html
```

Hasil 1 salah → masalah DNS/DHCP. Hasil 2 gagal → proxy. Hasil 3 gagal → backend.

---

## Langkah 6 - Jawaban Dua Pertanyaan Konsep

### 6a. Bagaimana DNS, reverse proxy, dan backend bekerja sama sebagai satu sistem

Ketiganya adalah **tiga lapisan tanggung jawab** yang bekerja berurutan:

| Komponen | Pertanyaan yang dijawab | Pada soal ini |
|---|---|---|
| **DHCP** (pendukung) | "Aku siapa, dan DNS mana yang harus kutanya?" | Aegir memberi IP, gateway, dan DNS `.12` |
| **DNS** | "Nama ini alamatnya di mana?" | Chernobog: `www.sekolah.local` → `.11` (Kazdel) |
| **Reverse proxy** | "Path ini harus diserahkan ke siapa?" | Kazdel: `/nilai`, `/absensi`, `/kalender` → web1/2/3 |
| **Backend** | "Apa isi aplikasinya?" | web1/web2/web3 menyajikan HTML masing-masing |
| **Router/NAT** | "Bagaimana keluar ke internet?" | Hanya untuk instalasi paket, bukan untuk jalur akses user |

Alur satu request `curl http://www.sekolah.local/nilai` dari Sargon:

```
Sargon                Chernobog(DNS)          Kazdel(Proxy)           web1(Backend)
  |-- 1. A? www.sekolah.local -->|                  |                      |
  |<-- 2. [IP Prefix].200.11 ----|                  |                      |
  |-- 3. TCP :80, GET /nilai, Host: www.sekolah.local ->|                  |
  |                                  | 4. cocokkan server_name + location |
  |                                  |-- 5. GET /web1.html (:8080) ------>|
  |                                  |<-- 6. 200 OK + HTML Nilai ---------|
  |<-- 7. 200 OK + HTML Nilai -------|                                    |
```

Inti kerja samanya:

1. **DNS memisahkan "nama" dari "alamat"** sehingga user memakai nama mudah diingat. Seluruh nama diarahkan ke **satu alamat**: reverse proxy.
2. **Reverse proxy memisahkan "alamat publik" dari "lokasi sebenarnya"**. Ia membaca header `Host` (virtual host) dan path (`location`) untuk memilih backend, lalu membuka koneksi **baru** ke backend dan menyalin jawabannya. Client tidak pernah tahu IP backend.
3. **Backend hanya fokus pada isi aplikasi**. Mereka bisa dipindah/ditambah/diganti tanpa mengubah URL yang dilihat user; cukup ubah `proxy_pass`.
4. **Keamanan dan keteraturan:** hanya satu titik masuk (port 80 di Kazdel) yang perlu dijaga/di-log/diberi TLS; backend bisa dibatasi (`allow` Kazdel saja). Header `X-Real-IP` menjaga informasi client asli.

### 6b. Apa yang terjadi bila salah satu komponen tidak berfungsi

| Komponen mati | Gejala di client | Yang masih berfungsi | Cara membuktikan / mendiagnosis |
|---|---|---|---|
| **DNS (Chernobog/named)** | `ping/curl www.sekolah.local` → `Could not resolve host` / `bad address`. Browser: "server not found" | Akses via IP (`curl http://[IP Prefix].200.11/nilai -H "Host: www.sekolah.local"`) tetap jalan; domain yang sudah ada di cache masih bisa sampai TTL habis; internet via `8.8.8.8` sekunder bisa menjawab nama publik tapi **tidak** nama internal | `dig @[IP Prefix].200.12 www.sekolah.local` timeout; `ps | grep named` |
| **Reverse proxy (Kazdel/nginx)** | `Connection refused` (nginx mati) atau `timed out` (node mati) pada `www.sekolah.local` | DNS tetap menjawab benar; backend sehat (bisa diakses langsung dari node yang diizinkan); **semua tiga aplikasi sekaligus tidak bisa diakses** karena single point of failure | `dig` OK, `curl http://[IP Prefix].200.11` gagal; `netstat -tlnp | grep :80` di Kazdel |
| **Satu backend (mis. web2)** | `/absensi` → `502 Bad Gateway` (atau `504` jika timeout); **hanya path itu** yang gagal | `/nilai` dan `/kalender` normal, DNS dan proxy tetap sehat | `error.log` Kazdel: `connect() failed (111: Connection refused)`; `curl http://[IP Prefix].200.21:8080/web2.html` dari Kazdel |
| **DHCP (Aegir)** | Client baru tidak mendapat IP (atau lease lama kedaluwarsa) | Server statis dan client yang masih punya lease tetap jalan; seluruh layanan (DNS, proxy, backend) normal bila IP sudah ada | `udhcpc` tidak mendapat OFFER; lease di `dhcpd.leases` |
| **Router/NAT** | Tidak ada internet (`ping 8.8.8.8` gagal) | Seluruh layanan **internal** (DNS, proxy, backend) tetap jalan karena semuanya satu subnet `.200.0/24` | `ping 8.8.8.8`, `iptables -t nat -L -n -v`, `ip_forward` |
| **Salah satu konfigurasi salah** (mis. record DNS salah / `proxy_pass` salah) | Nama ter-resolve ke IP salah → halaman salah/timeout; `proxy_pass` salah → 502/404 | Komponen lain normal | Cocokkan IP pada `dig` vs IP Kazdel; periksa `proxy_pass` |

Pelajaran penting: **status code menunjuk lapisan yang rusak**: gagal resolve → DNS; `refused/timeout` ke `.11` → proxy; `502/504` → backend atau jalur proxy→backend; `404` → routing path/konten.

---

## Cheat Sheet Debugging

### Urutan diagnosis (bawah ke atas)

| Lapisan | Pertanyaan | Perintah |
|---|---|---|
| Fisik/L2 | Link tersambung ke switch yang benar? | Lihat warna link GNS3, `ip link` |
| L3 | IP, gateway, rute benar? | `ip addr`, `ip route`, `ping <gw>` |
| Internet | NAT jalan? | `ping 8.8.8.8` dari node, `iptables -t nat -L -n -v` di Router |
| DNS | Nama ter-resolve? | `cat /etc/resolv.conf`, `dig @<dns> <nama>`, `nslookup` |
| L4 | Port listen/terjangkau? | `netstat -tlnp`, `curl -v --connect-timeout 3`, `nc -zv <ip> <port>` |
| L7 | Aplikasi menjawab benar? | `curl -v`, `error.log`, `access.log`, `named.log` |

### Perintah cepat

```bash
# Validasi konfigurasi
nginx -t -c /root/myconfig/nginx.conf
named-checkconf /root/dns/named.conf
named-checkzone sekolah.local /root/dns/db.sekolah.local
dhcpd -t -cf /etc/dhcp/dhcpd.conf

# Proses dan port
ps | grep -E "nginx|named|dhcpd"
netstat -tlnp ; netstat -ulnp
pkill nginx ; pkill named

# Uji end-to-end tanpa bergantung DNS
curl -v -H "Host: www.sekolah.local" http://[IP Prefix].200.11/nilai
curl --resolve www.sekolah.local:80:[IP Prefix].200.11 http://www.sekolah.local/absensi

# Tangkap paket
tcpdump -ni eth0 port 53 or port 80 or port 8080
```

| HTTP | Arti | Biasanya |
|---|---|---|
| 200 | OK | - |
| 403 | Forbidden | `allow/deny` backend, permission file |
| 404 | Not Found | Path/`location` tidak cocok, nama file salah |
| 502 | Bad Gateway | Backend mati/IP:port salah |
| 504 | Gateway Timeout | Backend tidak merespons / rute putus |

---

## Checklist Laporan

- [ ] Screenshot topologi GNS3 (semua node dan link, label IP)
- [ ] Router: `iptables -t nat -L -n -v`, `ping google.com`
- [ ] Aegir: `dhcpd.conf` (routers, domain-name-servers, domain-name)
- [ ] Client: `cat /etc/resolv.conf` (Sargon, Iberia, Higashi, Ognisko)
- [ ] Chernobog: `named.conf`, `db.sekolah.local`, `named.log` yang memuat query client
- [ ] web1/web2/web3: `nginx.conf`, file `webN.html`, `netstat -tlnp | grep 8080`
- [ ] Kazdel: `nginx.conf` reverse proxy
- [ ] `curl -v` untuk `/nilai`, `/absensi`, `/kalender` (satu domain, port 80)
- [ ] Screenshot browser Ognisko untuk ketiga URL
- [ ] `access.log` backend: sumber hanya IP Kazdel
- [ ] Jawaban 6a dan 6b (bagian "Langkah 6")
