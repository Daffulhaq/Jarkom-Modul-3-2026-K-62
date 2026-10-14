# Jarkom-Modul-3-2026-K-62

| Nama | NRP |
| :--- | :---: |
| Daffa Ulhaq Fadhlurrahman | 5027251033 |
| Michiko Artika Satriyo | 5027251105 |

## Soal 1.
Ketika Curtain menutup Shibuya, jalur komunikasi antar zona Jujutsu High terputus. Pulihkan konektivitas dasar dengan mengonfigurasi jichi: eth0 memperoleh WAN dari NAT, sedangkan eth1–eth4 menjadi gateway Network 1–4 sesuai tabel pemetaan. Aktifkan IPv4 forwarding dan source NAT, lalu tetapkan IP statis host infrastruktur sesuai tabel spesifikasi. Buktikan konektivitas setiap host ke gateway dan akses eksternal dari host yang relevan.

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