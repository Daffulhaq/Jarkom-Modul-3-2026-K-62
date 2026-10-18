# Jarkom-Modul-3-2026-K-62

| Nama | NRP |
| :--- | :---: |
| Daffa Ulhaq Fadhlurrahman | 5027251033 |
| Michiko Artika Satriyo | 5027251105 |

## Soal 1.
Ketika Curtain menutup Shibuya, jalur komunikasi antar zona Jujutsu High terputus. Pulihkan konektivitas dasar dengan mengonfigurasi jichi: eth0 memperoleh WAN dari NAT, sedangkan eth1–eth4 menjadi gateway Network 1–4 sesuai tabel pemetaan. Aktifkan IPv4 forwarding dan source NAT, lalu tetapkan IP statis host infrastruktur sesuai tabel spesifikasi. Buktikan konektivitas setiap host ke gateway dan akses eksternal dari host yang relevan.

### File-File Konfigurasi Setiap Nodes
[script config nodes](script-config.md)

### Testing
- `ping -c 2 8.8.8.8`
![dokumentasi](assets/1.testing-gugel.png)

- testing pada `jichi` dan `8.8.8.8`
![dokumentasi](assets/1.testing-jichi.png)

- testing node ke router
![dokumentasi](assets/1.utahime-to-jichi.png)

- testing antar nodes beda switch
![dokumentasi](assets/1.utahime-to-tengen.png)
![dokumentasi](assets/1.yaga-to-gojo.png)

## Soal 2.
Tengen, penjaga pusat distribusi di Data Center, harus mengambil alih pengalokasian alamat bagi wilayah yang membutuhkan konfigurasi dinamis. Konfigurasikan isc-dhcp-server pada [PREFIX IP].4.10. Deklarasikan Network 4 ([PREFIX IP].4.0/24) sebagai subnet lokal tanpa pool dinamis, lalu siapkan scope DHCP untuk Network 2 ([PREFIX IP].2.0/24) dan Network 3 ([PREFIX IP].3.0/24). Validasi konfigurasi dan pastikan service aktif tanpa error “No subnet declaration”.

### Testing
- validasi konfigurasi `tengen`
![dokumentasi](assets/2.validasi-conf.png)

- cek service aktif
![dokumentasi](assets/2.service-aktif.png)

- memastikan tanpa error
![dokumentasi](assets/2.no-subnet-dec.png)
![dokumentasi](assets/2.no-subnet-dec-bg.png)

## Soal 3.
Permintaan alamat dari Divisi A dan Divisi B tidak dapat menyeberangi subnet secara langsung. Konfigurasikan jichi sebagai DHCP Relay menuju tengen ([PREFIX IP].4.10), menggunakan eth2 untuk Network 2 dan eth3 untuk Network 3. Buktikan bahwa client dari kedua subnet berhasil memperoleh lease melalui relay.

### Pengerjaan
Menjalankan relay di `jichi`:
`pkill dhcrelay`
`/usr/sbin/dhcrelay -i eth2 -i eth3 -i eth4 192.242.4.10`
`pgrep -a dhcrelay`

Memastikan `tengen` aktif (menggunakan command `pgrep -a dhcpd`)

### Testing
- `itadori`:
![dokumentasi](assets/3.testing-itadori.png)

- `tengen`cek entri leases dengan `cat /var/lib/dhcp/dhcpd.leases`

## Soal 4.
Agar tim Dev dan Support di Divisi A menerima jatah alamat yang berbeda, atur dua pool DHCP pada Network 2. Tetapkan Pool 1 ([PREFIX IP].2.10–[PREFIX IP].2.35) untuk itadori dan fushiguro, serta Pool 2 ([PREFIX IP].2.70–[PREFIX IP].2.95) untuk zenin dan okkotsu. Rentang [PREFIX IP].2.36–[PREFIX IP].2.69 tidak boleh dialokasikan. Karena seluruh client berada pada subnet yang sama, gunakan DHCP class berdasarkan MAC Address atau Client ID untuk memisahkan kedua kelompok.

### Pengerjaan
Setiap client memakai MAC Address tetap agar bisa dikenali oleh class (diatur pada script client):

| Client | MAC Address |
| :--- | :--- |
| itadori | `de:ad:be:ef:62:02` |
| fushiguro | `de:ad:be:ef:62:03` |
| zenin | `de:ad:be:ef:62:04` |
| okkotsu | `de:ad:be:ef:62:05` |

Konfigurasi untuk Soal 4 (class berdasarkan MAC dengan `match hardware`, lalu dua pool pada subnet Network 2):

```
class "pool1" { match hardware; }
subclass "pool1" 1:de:ad:be:ef:62:02;   # itadori
subclass "pool1" 1:de:ad:be:ef:62:03;   # fushiguro

class "pool2" { match hardware; }
subclass "pool2" 1:de:ad:be:ef:62:04;   # zenin
subclass "pool2" 1:de:ad:be:ef:62:05;   # okkotsu

subnet 192.242.2.0 netmask 255.255.255.0 {
    option routers 192.242.2.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.242.2.255;

    pool {
        range 192.242.2.10 192.242.2.35;
        allow members of "pool1";
    }
    pool {
        range 192.242.2.70 192.242.2.95;
        allow members of "pool2";
    }
}
```

Angka `1:` pada subclass adalah hardware type Ethernet. Rentang `.36–.69` tidak dimasukkan ke `range` mana pun sehingga tidak pernah dialokasikan, dan client dengan MAC di luar daftar tidak masuk class sehingga tidak mendapat IP.

### Testing
Pada masing-masing client (`itadori`, `fushiguro`, `zenin`, `okkotsu`), buang lease lama agar proses DORA berjalan penuh, lalu minta lease baru:

```bash
pkill dhclient
ip addr flush dev eth0
rm -f /var/lib/dhcp/dhclient*.leases
dhclient -v eth0
ip -br addr show eth0
ip route
```
- `itadori` 
<img width="756" height="417" alt="4 lease-itadori" src="https://github.com/user-attachments/assets/624f0694-e9f6-4c66-962f-1d51135147f8" />

- `fushiguro` 
<img width="758" height="637" alt="4 lease-fushiguro" src="https://github.com/user-attachments/assets/56caa5ac-58ff-4545-be5d-28d89a65e2e4" />

- `zenin` 
<img width="761" height="675" alt="4 lease-zenin" src="https://github.com/user-attachments/assets/72365eb9-eb9f-4a58-b12d-617746a22378" />

- `okkotsu` 
<img width="760" height="536" alt="4 lease-okkotsu" src="https://github.com/user-attachments/assets/6ae106e2-51a3-4d1a-9743-aaeafe77b415" />

- Pada `tengen`, ringkasan lease (IP, MAC, status):
`awk '/^lease/{ip=$2} $1=="binding"{st=$3} $1=="hardware"{mac=$3} /^}/{print ip, mac, st}' /var/lib/dhcp/dhcpd.leases | sort -u`
<img width="768" height="162" alt="4 ip-mac-status" src="https://github.com/user-attachments/assets/88968944-e248-4a2a-ad00-7347bceff3e8" />

- Pada `tengen`, membuktikan rentang `.2.36–.2.69` tidak boleh dialokasikan (output harus kosong):
`awk '/^lease/{print $2}' /var/lib/dhcp/dhcpd.leases | grep -E '^192\.242\.2\.(3[6-9]|[4-6][0-9])$'`
![dokumentasi](assets/4.range-terlarang.png)

## Soal 5.
Setiap tim lapangan memiliki durasi operasi yang berbeda. Pada Network 3, berikan short lease 30 menit (1800 detik) kepada ino dan kusakabe, serta long lease 12 jam (43200 detik) kepada miwa. Gunakan pool short lease [PREFIX IP].3.10–[PREFIX IP].3.89 dan pool long lease [PREFIX IP].3.100–[PREFIX IP].3.199. Sesuaikan default-lease-time dan max-lease-time untuk setiap kelas client. Berikan pula DHCP options berupa DNS [PREFIX IP].1.2 dan [PREFIX IP].1.3, serta domain-search <xxxx>.com.

### Pengerjaan
Pada `dhcpd.conf` di `tengen`, DHCP options (DNS dan domain-search) diletakkan di bagian global sehingga berlaku untuk Network 2 maupun Network 3:

```
authoritative;
default-lease-time 600;
max-lease-time 7200;
option domain-name-servers 192.242.1.2, 192.242.1.3;
option domain-search "K62.com";
```

Class untuk tiap kelompok lease:

| Client | MAC Address | Class |
| :--- | :--- | :--- |
| ino | `de:ad:be:ef:62:06` | short-lease |
| kusakabe | `de:ad:be:ef:62:07` | short-lease |
| miwa | `de:ad:be:ef:62:08` | long-lease |

```
class "short-lease" {
    match hardware;
    default-lease-time 1800;
    max-lease-time 1800;
}
subclass "short-lease" 1:de:ad:be:ef:62:06;   # ino
subclass "short-lease" 1:de:ad:be:ef:62:07;   # kusakabe

class "long-lease" {
    match hardware;
    default-lease-time 43200;
    max-lease-time 43200;
}
subclass "long-lease" 1:de:ad:be:ef:62:08;    # miwa
```

Subnet Network 3 dengan dua pool:

```
subnet 192.242.3.0 netmask 255.255.255.0 {
    option routers 192.242.3.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.242.3.255;

    pool {
        range 192.242.3.10 192.242.3.89;
        allow members of "short-lease";
    }
    pool {
        range 192.242.3.100 192.242.3.199;
        allow members of "long-lease";
    }
}
```

### Testing
Pada `ino`, `kusakabe`, dan `miwa`, minta lease baru:

```bash
pkill dhclient
ip addr flush dev eth0
rm -f /var/lib/dhcp/dhclient*.leases
dhclient -v eth0
ip -br addr show eth0
ip route
cat /etc/resolv.conf
```

- `ino` 
<img width="766" height="510" alt="5 lease-ino" src="https://github.com/user-attachments/assets/d4e6ff83-c58d-43b6-8810-4276d2308fe3" />

- `kusakabe` 
<img width="761" height="515" alt="5 lease-kusakabe" src="https://github.com/user-attachments/assets/4fdfdcdb-bfcc-4460-b92d-13b1ae79087e" />

- `miwa` 
<img width="766" height="511" alt="5 lease-miwa" src="https://github.com/user-attachments/assets/7df44f8f-f92b-40ba-8539-e5e3fa9d16ba" />

Kemudian, cek bukti lease time di `tengen`

```
grep -E "^lease|starts|ends|hardware" /var/lib/dhcp/dhcpd.leases
```

<img width="762" height="345" alt="5 lease-time" src="https://github.com/user-attachments/assets/4fe3fcee-fafa-4e38-8ca7-472caaca8763" />


## Soal 6.
Kugisaki ditetapkan sebagai personel VIP yang harus mempertahankan alamat yang sama setiap kali terhubung. Buat DHCP static lease berdasarkan MAC Address client tersebut dengan fixed address [PREFIX IP].3.99. Pastikan alamat ini berada di luar seluruh pool dinamis, validasi konfigurasi server, lalu renew lease pada client. Buktikan bahwa alamat yang diterima sesuai dengan reservasi.

### Testing
- Pada `kugisaki`, minta lease baru:
```bash
pkill dhclient
ip addr flush dev eth0
rm -f /var/lib/dhcp/dhclient*.leases
dhclient -v eth0
```
<img width="763" height="417" alt="6 lease-kugisaki" src="https://github.com/user-attachments/assets/7df6806f-b574-4114-aede-15157e09aa64" />

- DI `tengen`, kita perlu bukti DHCP penuh untuk `kugisaki`:
```bash
tcpdump -l -n -vvv -i eth0 '(udp port 67 or udp port 68) and udp[36:4]=0xdeadbeef and udp[40:2]=0x6201' | grep -E "Request from|Reply|Gateway-IP|Your-IP|Message \(53\)|Requested-IP|Lease-Time"
```
<img width="762" height="796" alt="6 DHCP-static-lease" src="https://github.com/user-attachments/assets/d16d3d56-3b61-4a48-aefd-ae7d9eb1002c" />

## Soal 7.
Setelah aturan alokasi diterapkan, pastikan setiap client menerima konfigurasi jaringan sesuai wilayahnya: subnet mask 255.255.255.0 (/24), gateway dan broadcast address yang benar, DNS [PREFIX IP].1.2 serta [PREFIX IP].1.3, dan domain-search yang ditentukan. Periksa IP address dan routing seluruh client Divisi A dan Divisi B, kemudian cocokkan lease dengan /var/lib/dhcp/dhcpd.leases pada tengen. Pastikan tidak ada alamat yang bertabrakan atau berada di luar kebijakan pool.

### Pengerjaan
Seluruh nilai yang diterima client berasal dari `/etc/dhcp/dhcpd.conf` di `tengen` (soal 2 sampai soal 6):
| Nilai | Sumber di `dhcpd.conf` |
| :--- | :--- |
| DNS `192.242.1.2`, `192.242.1.3` | `option domain-name-servers` (global) |
| Domain-search `K62.com` | `option domain-search` (global) |
| Subnet mask, gateway, broadcast | `option subnet-mask`, `option routers`, `option broadcast-address` pada tiap blok `subnet` |

### Testing
- Tampilkan IP, mask, broadcast, gateway, DNS, domain-search, dan optiondari server untuk setiap node di switch 5, 6, 7, dan 8:
```bash
ip -4 addr show eth0
ip route
cat /etc/resolv.conf
grep -E 'fixed-address|subnet-mask|routers|broadcast-address|domain-name-servers|domain-search' /var/lib/dhcp/dhclient*.leases
```
| Client | MAC | IP/Mask | Gateway | Broadcast | DNS | Domain-search |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| itadori | `de:ad:be:ef:62:02` | `192.242.2.10/24` | `192.242.2.1` | `192.242.2.255` | `192.242.1.2`, `192.242.1.3` | `K62.com.` |
| fushiguro | `de:ad:be:ef:62:03` | `192.242.2.11/24` | `192.242.2.1` | `192.242.2.255` | `192.242.1.2`, `192.242.1.3` | `K62.com.` |
| zenin | `de:ad:be:ef:62:04` | `192.242.2.70/24` | `192.242.2.1` | `192.242.2.255` | `192.242.1.2`, `192.242.1.3` | `K62.com.` |
| okkotsu | `de:ad:be:ef:62:05` | `192.242.2.71/24` | `192.242.2.1` | `192.242.2.255` | `192.242.1.2`, `192.242.1.3` | `K62.com.` |
| ino | `de:ad:be:ef:62:06` | `192.242.3.10/24` | `192.242.3.1` | `192.242.3.255` | `192.242.1.2`, `192.242.1.3` | `K62.com.` |
| kusakabe | `de:ad:be:ef:62:07` | `192.242.3.11/24` | `192.242.3.1` | `192.242.3.255` | `192.242.1.2`, `192.242.1.3` | `K62.com.` |
| miwa | `de:ad:be:ef:62:08` | `192.242.3.100/24` | `192.242.3.1` | `192.242.3.255` | `192.242.1.2`, `192.242.1.3` | `K62.com.` |
| kugisaki | `de:ad:be:ef:62:01` | `192.242.3.99/24` | `192.242.3.1` | `192.242.3.255` | `192.242.1.2`, `192.242.1.3` | `K62.com.` |

- Pada `tengen`, daftar lease aktif untuk dicocokkan dengan IP di client:
![dokumentasi](assets/7.lease-aktif.png)

| IP | MAC | Client | Pool |
| :--- | :--- | :--- | :--- |
| `192.242.2.10` | `de:ad:be:ef:62:02` | itadori | Pool 1 (`.2.10–.2.35`) |
| `192.242.2.11` | `de:ad:be:ef:62:03` | fushiguro | Pool 1 (`.2.10–.2.35`) |
| `192.242.2.70` | `de:ad:be:ef:62:04` | zenin | Pool 2 (`.2.70–.2.95`) |
| `192.242.2.71` | `de:ad:be:ef:62:05` | okkotsu | Pool 2 (`.2.70–.2.95`) |
| `192.242.3.10` | `de:ad:be:ef:62:06` | ino | Short lease (`.3.10–.3.89`) |
| `192.242.3.11` | `de:ad:be:ef:62:07` | kusakabe | Short lease (`.3.10–.3.89`) |
| `192.242.3.100` | `de:ad:be:ef:62:08` | miwa | Long lease (`.3.100–.3.199`) |

- `kugisaki` tidak muncul di `dhcpd.leases` karena alamat `fixed-address` tidak dicatat di file tersebut. Reservasinya dibuktikan dari sisi client (IP `192.242.3.99`) dan dari konfigurasi di `tengen`:
![dokumentasi](assets/7.ip-kugisaki.png)
![dokumentasi](assets/7.conf-kugisaki.png)

- Pada `tengen`, membuktikan tidak ada alamat di luar kebijakan pool (output harus kosong):
![dokumentasi](assets/7.cek-IP-diluar-pool.png)

- Membuktikan tidak ada IP yang bertabrakan di antara 8 client (output harus kosong):
![dokumentasi](assets/7.IP-tabrak.png)

## Soal 8.
Sebelum Curtain dibuka kembali, buktikan bahwa proses pemberian alamat berjalan sesuai protokol. Lakukan packet capture saat salah satu client Network 2 atau Network 3 meminta lease baru. Gunakan tcpdump pada tengen atau jalur menuju Data Center untuk menunjukkan urutan DHCPDISCOVER → DHCPOFFER → DHCPREQUEST → DHCPACK. Identifikasi alamat server, alamat yang diberikan, lease time, dan field giaddr. Pastikan nilai giaddr sesuai dengan interface jichi yang menerima request dari subnet client.

### Testing
- Pada `itadori`, buang lease lama agar proses DORA berjalan, lalu minta lease baru:
```bash
pkill dhclient
ip addr flush dev eth0
rm -f /var/lib/dhcp/dhclient*.leases
dhclient -v eth0
```
![dokumentasi](assets/8.capture.png)

| Paket | Arah | Isi penting |
| :--- | :--- | :--- |
| DHCPDISCOVER | `192.242.4.1` → `192.242.4.10` | `Request from de:ad:be:ef:62:02`, `hops 1`, `Gateway-IP 192.242.2.1`, tanpa `Requested-IP` |
| DHCPOFFER | `192.242.4.10` → `192.242.2.1` | `Your-IP 192.242.2.10`, `Server-ID 192.242.4.10`, `Lease-Time 600` |
| DHCPREQUEST | `192.242.4.1` → `192.242.4.10` | `Requested-IP 192.242.2.10`, `Server-ID 192.242.4.10`, `Gateway-IP 192.242.2.1` |
| DHCPACK | `192.242.4.10` → `192.242.2.1` | `Your-IP 192.242.2.10`, `Server-ID 192.242.4.10`, `Lease-Time 600` |

Paket dari `jichi` ke `tengen` bersumber `192.242.4.1` (interface `eth4`, menghadap Data Center). Balasan `tengen` dikirim ke `192.242.2.1`, yaitu nilai `giaddr`, sehingga kembali ke `jichi` dan diteruskan ke client.

Identifikasi hasil capture:

| Informasi | Nilai | Sumber di capture |
| :--- | :--- | :--- |
| Alamat server | `192.242.4.10` | `Server-ID` pada Offer, Request, dan ACK |
| Alamat yang diberikan | `192.242.2.10` | `Your-IP` pada Offer dan ACK (di dalam Pool 1 `.2.10–.2.35`) |
| Lease time | `600` detik | `Lease-Time` pada Offer dan ACK (default global) |
| giaddr | `192.242.2.1` | `Gateway-IP` pada keempat paket |

- Pada `jichi`, membuktikan `giaddr` sama dengan IP interface yang menerima request dari Network 2:
![dokumentasi](assets/8.giaddr-jichi.png)

`eth2` bernilai `192.242.2.1/24`, sama dengan `Gateway-IP` pada capture. Untuk client Network 3, `giaddr` akan bernilai `192.242.3.1` (interface `eth3`).

## Soal 9.
Sebuah gangguan memutus jalur relay dan membuat permintaan alamat tidak sampai ke Data Center. Hentikan service DHCP Relay pada jichi, lalu paksa salah satu client di Network 2 atau Network 3 meminta lease baru dan catat gejalanya. Pulihkan service relay, ulangi permintaan DHCP, lalu buktikan bahwa client kembali memperoleh alamat dari pool yang benar dengan gateway semula.

### Pengerjaan
Kondisi normal sebelum gangguan pada `itadori`:
![dokumentasi](assets/9.itadori-awal.png)

Menghentikan relay pada `jichi`:
`pkill dhcrelay`
`pgrep -a dhcrelay`

Memulihkan relay pada `jichi`:
`/usr/sbin/dhcrelay -i eth2 -i eth3 -i eth4 192.242.4.10`
`pgrep -a dhcrelay`

### Testing
Pada `itadori`, buang lease lama agar permintaan DHCP dimulai dari awal, lalu minta lease baru.
- Saat relay mati, `dhclient` hanya mengirim `DHCPDISCOVER` berulang (6 kali dengan interval membesar), lalu berakhir dengan `No DHCPOFFERS received` dan `No working leases in persistent database - sleeping`. `eth0` tidak memiliki IPv4 dan `ip route` kosong.
![dokumentasi](assets/9.itadori-gagal.png)

- Setelah relay dipulihkan, urutan `DHCPDISCOVER`, `DHCPOFFER`, `DHCPREQUEST`, `DHCPACK` berjalan lengkap dan client kembali mendapat `192.242.2.10` (Pool 1) dengan `default via 192.242.2.1`, sama seperti sebelum gangguan.
![dokumentasi](assets/9.sesudah-itadori.png)

Menjalankan tcpdumb pada `jichi` dan `tengen`:
![dokumentasi](assets/9.tcpdumb-jichi.png)
![dokumentasi](assets/9.tcpdumb-tengen.png)

Kesimpulan: tanpa relay, DHCPDISCOVER broadcast dari Network 2 dan 3 berhenti di `jichi` dan tidak pernah mencapai `tengen`. Setelah relay dipulihkan, proses DHCP kembali normal dengan pool dan gateway yang benar.

## Soal 10.
Setelah gangguan Shibuya tertangani, pastikan pemulihan bertahan setelah restart. Aktifkan autostart DHCP Server pada tengen dan DHCP Relay pada jichi, lalu restart kedua node. Verifikasi status layanan, IP, gateway, lease, dan konektivitas client Network 2–3. Pastikan kebijakan Pool 1, Pool 2, short lease, long lease, dan fixed address tetap berlaku.

### Pengerjaan
Membuat script austostart untuk `jichi` dan `tengen`:
- isi script `soal10.sh` pada `jichi`:
``` bash
#!/bin/bash
for i in 1 2 3 4 5; do
    bash /root/jichi.sh > /root/autostart.log 2>&1
    pgrep -x dhcrelay >/dev/null && break
    sleep 15
done
```
- isi script `soal10.sh` pada `tengen`:
``` bash
#!/bin/bash
for i in 1 2 3 4 5; do
    bash /root/tengen.sh > /root/autostart.log 2>&1
    pgrep -x dhcpd>/dev/null && break
    sleep 15
done
```

### Testing
Setelah restart, periksa pada `jichi` dan `tengen`:
![dokumentasi](assets/10.jichi.png)
![dokumentasi](assets/10.tengen.png)

Pada setiap client Network 2 dan 3 (`itadori`, `fushiguro`, `zenin`, `okkotsu`, `ino`, `kusakabe`, `miwa`, `kugisaki`), minta lease baru dan uji konektivitas. Ping gateway disesuaikan dengan jaringan client (`192.242.2.1` untuk Network 2, `192.242.3.1` untuk Network 3):
```bash
pkill dhclient
ip addr flush dev eth0
rm -f /var/lib/dhcp/dhclient*.leases
dhclient -v eth0
ip -br addr show eth0
ip route
grep dhcp-lease-time /var/lib/dhcp/dhclient*.leases
ping -c 2 192.242.2.1    #(untuk Network 3: 192.242.3.1)
ping -c 2 8.8.8.8
```

Ringkasan hasil, menunjukkan kebijakan pool, lease, dan fixed address tetap berlaku setelah restart:

| Client | IP | Kebijakan | Gateway | Lease time pada client | Ping 8.8.8.8 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| itadori | `192.242.2.10` | Pool 1 (`.2.10–.2.35`) | `192.242.2.1` | 589 | 0% loss |
| fushiguro | `192.242.2.11` | Pool 1 (`.2.10–.2.35`) | `192.242.2.1` | 491 | 0% loss |
| zenin | `192.242.2.70` | Pool 2 (`.2.70–.2.95`) | `192.242.2.1` | 501 | 0% loss |
| okkotsu | `192.242.2.71` | Pool 2 (`.2.70–.2.95`) | `192.242.2.1` | 510 | 0% loss |
| ino | `192.242.3.10` | Short lease (`.3.10–.3.89`) | `192.242.3.1` | 1760 | 0% loss |
| kusakabe | `192.242.3.11` | Short lease (`.3.10–.3.89`) | `192.242.3.1` | 1766 | 0% loss |
| miwa | `192.242.3.100` | Long lease (`.3.100–.3.199`) | `192.242.3.1` | 43181 | 0% loss |
| kugisaki | `192.242.3.99` | Fixed address | `192.242.3.1` | 600 | 0% loss |

## Soal 11
Utahime dan Shoko ditugaskan membangun sistem penamaan domain resmi agar seluruh pos tidak perlu menghafal alamat IP mentah. Konfigurasikan BIND9 pada utahime sebagai DNS Master dan shoko sebagai DNS Slave untuk mengelola domain jujutsu.<xxxx>.com. Atur SOA record, NS delegasi untuk kedua server, dan terapkan allow-transfer dari Master ke Slave. Pastikan file zona berhasil tersinkronisasi otomatis ke shoko saat nomor serial di utahime dinaikkan, lalu buktikan dengan melakukan query dig ke kedua server dari salah satu client.

### Pengerjaan

Untuk pengerjaan soal ini, jalankan `master.sh` di utahime dan `slave.sh` di shoko.

### Testing
- Cek master (`utahime`):
```bash
dig @192.242.1.2 jujutsu.K62.com NS +noall +answer
dig @192.242.1.2 jujutsu.K62.com SOA +noall +answer
```
![dokumentasi](assets/11.bind9-aktif-utahime.png)

- Cek slave (`shoko`):
```bash
ls -l /var/lib/bind/
dig @192.242.1.3 jujutsu.K62.com SOA +noall +answer
```
![dokumentasi](assets/11.bind9-aktif-shoko.png)
![dokumentasi](assets/11.zone-file-slave.png)

- Uji sinkronisasi serial:
```bash
# Di utahime:
sed -i 's/2024101401/2024101402/' /etc/bind/jarkom/jujutsu.K62.com
named-checkzone jujutsu.K62.com /etc/bind/jarkom/jujutsu.K62.com
service named restart

# Di shoko:
dig @192.242.1.3 jujutsu.K62.com SOA +noall +answer
```
![dokumentasi](assets/11.sinkronisasi-serial.png)

## Soal 12
Peta domain harus memetakan pintu masuk utama dan tetap bisa mencari alamat di luar kawasan Shibuya. Pada utahime, buat A record jujutsu.<xxxx>.com yang mengarah ke gojo, serta CNAME www yang merujuk ke domain utama tersebut. Buat pula Reverse DNS (zona PTR) untuk melacak IP gojo kembali ke domain resminya, dan aktifkan fitur DNS Forwarder pada BIND9 mengarah ke IP gateway internet (192.168.122.1). Uji dari client dengan dig untuk mengecek CNAME, PTR, dan buktikan client tetap bisa melakukan ping ke domain internet seperti google.com.

### Testing 

Testing dilakukan di node client (contohnya `itadori`)
```bash
cat /etc/resolv.conf
dig jujutsu.K62.com +noall +answer
dig www.jujutsu.K62.com +noall +answer
dig -x 192.242.1.4 +noall +answer
dig google.com +short
ping -c 3 google.com
```
![dokumentasi](assets/12.dig-a-record.png)
![dokumentasi](assets/12.dig-cname.png)
![dokumentasi](assets/12.dig-ptr.png)
![dokumentasi](assets/12.ping-google.png)

## Soal 13
Kepala Sekolah Yaga dan Inumaki mendirikan database terpusat yang redundan agar log aktivitas kutukan tetap aman tersimpan. Konfigurasikan MariaDB pada yaga sebagai DB-Master dan inumaki sebagai DB-Slave. Buat database db_shibuya_<xxxx> beserta tabel request_logs di Master, aktifkan binary logging (log-bin, server-id), lalu hubungkan inumaki sebagai Slave yang menyalin data secara realtime dengan CHANGE MASTER TO. Berikan hak akses remote bagi seluruh node di Network 4, lalu buktikan replikasi berhasil dengan memasukkan data baru di Master dan memeriksa hasilnya di Slave.

### Pengerjaan
Jalankan master `yaga` terlebih dahulu, lalu slave `inumaki`:

```bash
# Di yaga:
./yaga.sh

# Di inumaki:
./inumaki.sh
```

### Testing
- Di `yaga` (master):
```bash
mysql -e "SHOW MASTER STATUS\G"
mysql -e "USE db_shibuya_K62; SHOW TABLES;"
mysql -e "SELECT user, host FROM mysql.user WHERE user IN ('shibuya','repl');"
```
![dokumentasi](assets/13.master-status.png)

- Di `inumaki` (slave):
```bash
mysql -e "SHOW SLAVE STATUS\G" | grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind_Master|Last_.*Error"
```
![dokumentasi](assets/13.slave-status.png)

- Uji replikasi:
```bash
# Di yaga:
mysql -e "USE db_shibuya_K62; INSERT INTO request_logs (worker, endpoint, client_ip) VALUES ('uji-master','/test','192.242.9.9');"

# Di inumaki:
mysql -e "SELECT * FROM db_shibuya_K62.request_logs ORDER BY id DESC LIMIT 3;"
```
![dokumentasi](assets/13.replikasi-bukti.png)

- Uji hak akses remote (dari node lain):
```bash
apt-get update && apt-get install -y mariadb-client
mysql -h 192.242.4.11 -ushibuya -pshibuya123 -e "SELECT COUNT(*) FROM db_shibuya_K62.request_logs;"
```
![dokumentasi](assets/13.remote-access.png)

## Soal 14
Para penyihir lapangan bersiaga di Data Center sebagai pekerja backend untuk memproses aplikasi. Pasang Nginx dan PHP-FPM pada maki dan panda untuk menyajikan web frontend yang menampilkan hostname masing-masing. Pada nanami, todo, dan kirara, siapkan layanan API yang mencatat log request ke database yaga. Konfigurasikan virtual host agar worker merespons domain layanan yang ditentukan. Batasi akses langsung dari klien ke worker, web worker hanya menerima trafik melalui jalur proxy web yang ditentukan, sedangkan worker API hanya menerima trafik melalui geto. Uji bahwa akses langsung dari klien ditolak, sementara akses melalui reverse proxy tetap berhasil.

### Pengerjaan
Jalankan file config pada node `maki`, `panda`, `nanami`, `todo`, dan `kirara` sebelum melakukan testing.

### Testing
- Akses lokal:
```bash
# Di maki, panda, nanami, todo, kirara:
curl -s http://127.0.0.1
```
![dokumentasi](assets/14.worker-lokal.png)
![dokumentasi](assets/14.api-lokal.png)

- Akses langsung dari client:
```bash
# DI itadori:
curl -s -o /dev/null -w "%{http_code}\n" http://192.242.4.21  
curl -s -o /dev/null -w "%{http_code}\n" http://192.242.4.23  
```
![dokumentasi](assets/14.direct-denied.png)

## Soal 15
Geto bertugas mengendalikan pembagian beban request API di internal Data Center agar server tidak tumbang bersamaan. Konfigurasikan Nginx pada geto sebagai Load Balancer internal dengan upstream api_cluster yang mengarahkan trafik ke nanami dan todo. Terapkan algoritma Weighted Round Robin dengan bobot 3:1, di mana nanami menerima porsi beban tiga kali lebih banyak daripada todo. Buktikan distribusi beban ini dengan melakukan loop request berulang kali ke geto dari jaringan lokal Data Center dan periksa rasio responnya.


### Pengerjaan
Jalankan file `geto.sh` di node geto.

### Testing
Dari node Network 4 (misal yaga):

```bash
apt-get install -y curl 2>/dev/null
for i in $(seq 1 40); do
  curl -s http://192.242.4.20/ | grep -o '"worker":"[a-z]*"'
done | sort | uniq -c
```
![dokumentasi](assets/15.wrr-geto.png)

## Soal 16
Gojo berdiri di perbatasan DMZ sebagai gerbang publik tunggal bagi seluruh client dari luar maupun dalam. Konfigurasikan Nginx pada gojo untuk melayani domain jujutsu.<xxxx>.com dan www.jujutsu.<xxxx>.com menggunakan metode Path-Based Routing. Atur agar request root (/) diteruskan secara Round Robin ke worker frontend (maki dan panda), sedangkan request ke path /api diteruskan langsung ke geto. Uji dari client Divisi A atau B bahwa akses ke web utama dibalas oleh Maki/Panda, sedangkan akses ke /api diteruskan dan dijawab oleh Nanami/Todo.

### Pengerjaan
Jalankan file `soal16.sh` pada node gojo.

### Testing
Dari client:

```bash
apt-get update && apt-get install -y curl dnsutils

# Round Robin maki/panda
for i in $(seq 1 20); do
  curl -s "http://jujutsu.K62.com/?nocache=$RANDOM"
  echo
done | grep -oi "Hostname: [a-z]*" | sort | uniq -c

# /api -> geto
for i in $(seq 1 4); do
  curl -s "http://jujutsu.K62.com/api"
  echo
done
```
![dokumentasi](assets/16.gojo-rr-frontend.png)
![dokumentasi](assets/16.gojo-api-geto.png)

## Soal 17
Komando pusat perlu mengetahui IP asli klien yang mengirimkan request tanpa tertukar dengan IP proxy perantara. Konfigurasikan gojo dan geto agar meneruskan informasi alamat klien melalui header X-Real-IP dan X-Forwarded-For. Pastikan geto hanya mempercayai informasi alamat klien yang diterima dari proxy terpercaya, dan skrip API menyimpan alamat klien yang telah diverifikasi ke kolom client_ip pada tabel request_logs di database yaga. Uji dengan mengirimkan request dari klien Network 2 dan Network 3 ke endpoint /api, kemudian periksa database untuk memastikan alamat yang tercatat sesuai dengan IP klien asal, bukan IP gojo atau geto.

### Testing

```bash
# Di itadori:
curl -s "http://jujutsu.K62.com/api"; echo

# Di ino:
curl -s "http://jujutsu.K62.com/api"; echo

# Di yaga:
mysql -e "SELECT id, worker, client_ip, created_at FROM db_shibuya_K62.request_logs ORDER BY id DESC LIMIT 6;"
```
![dokumentasi](assets/17.api-request-clients.png)
![dokumentasi](assets/17.db-client-ip.png)
