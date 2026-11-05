# catatan

<img width="706" height="862" alt="image" src="https://github.com/user-attachments/assets/579422b0-8d5d-449a-a4d1-82cb4f4f7401" />


0. Sebelum mulai
Tutup Chrome dan aplikasi berat lain (RAM yang penuh sudah membuat GNS3 crash berkali-kali).
Nyalakan GNS3 VM, tunggu sampai muncul layar dengan IP.
Buka GNS3, buka project, dan pastikan Servers Summary menampilkan gns3vm berlampu hijau.
Start node satu per satu, jangan Start all. Urutannya: RhodesIsland, Lungmen, Ursus, Yan, Victoria, Laterano, Columbia.
Jangan klik berulang saat GNS3 lag.



1. RhodesIsland (router utama)
Cek IP. Jalankan ip -br addr. Harus ada:

eth0 dapat IP DHCP dari NAT (misalnya 192.168.122.x)
eth1 = 10.239.1.1/24
eth2 = 10.239.2.1/24

Kalau eth1 atau eth2 kosong, edit /etc/network/interfaces:
auto eth0
iface eth0 inet dhcp
auto eth1
iface eth1 inet static
address 10.239.1.1
netmask 255.255.255.0
auto eth2
iface eth2 inet static
address 10.239.2.1
netmask 255.255.255.0


Lalu restart interface atau node.


Route, NAT, dan DNS. Tiga hal ini hilang setiap node restart, jadi simpan sebagai /root/init.sh:
cat > /root/init.sh << 'EOF'
ip route add 10.239.3.0/24 via 10.239.1.2 dev eth1
ip route add 10.239.4.0/24 via 10.239.2.2 dev eth2
iptables -t nat -A POSTROUTING -j MASQUERADE -s 10.239.0.0/16 -o eth0
echo "nameserver 8.8.8.8" > /etc/resolv.conf
ip route show
EOF
chmod +x /root/init.sh
sh /root/init.sh

Error “File exists” itu aman (route sudah ada).

Verifikasi:
- ip route show harus menampilkan 4 subnet (1.0, 2.0, 3.0, 4.0) plus default via eth0.
- ping -c 3 google.com harus berhasil.
- tc qdisc show dev eth1 dan eth2 harus hanya fq_codel, tanpa netem atau tbf.



2. Lungmen
/etc/network/interfaces:
```
auto eth0
iface eth0 inet static
address 10.239.1.2
netmask 255.255.255.0
gateway 10.239.1.1
auto eth1
iface eth1 inet static
address 10.239.3.1
netmask 255.255.255.0
```


/root/init.sh:
```
ip route add 10.239.2.0/24 via 10.239.1.1 dev eth0
ip route add 10.239.4.0/24 via 10.239.1.1 dev eth0
echo "nameserver 8.8.8.8" > /etc/resolv.conf
ip route show
```
Jalankan sh /root/init.sh. Hasilnya harus 4 subnet dan ping -c 2 google.com berhasil.



3. Ursus
/etc/network/interfaces:
```
auto eth0
iface eth0 inet static
address 10.239.2.2
netmask 255.255.255.0
gateway 10.239.2.1
auto eth1
iface eth1 inet static
address 10.239.4.1
netmask 255.255.255.0
```
/root/init.sh:
```
ip route add 10.239.1.0/24 via 10.239.2.1 dev eth0
ip route add 10.239.3.0/24 via 10.239.2.1 dev eth0
echo "nameserver 8.8.8.8" > /etc/resolv.conf
ip route show
```
Verifikasinya sama seperti Lungmen: 4 subnet dan ping google berhasil.

Tes jalur router. Dari RhodesIsland jalankan ping -c 2 10.239.3.1 dan ping -c 2 10.239.4.1. Keduanya harus balas.



4. Yan (DHCP subnet 4.0)
Cek `ip -br addr: eth0` = 10.239.4.2/24. Tambahkan `ip route add default via 10.239.4.1` kalau belum ada gateway, dan echo `"nameserver 8.8.8.8" > /etc/resolv.conf`.
Pastikan internet jalan (ping google.com), karena Laterano butuh Yan dan internet untuk install paket.
Ambil MAC eth0 Laterano dulu (`ip link show eth0` di Laterano). Contoh milikmu: 02:42:1c:c1:e1:00.
Tulis /etc/kea/kea-dhcp4.conf (hapus isi lama dulu):
```
{
"Dhcp4": {
"interfaces-config": { "interfaces": ["eth0"] },
"lease-database": { "type": "memfile", "persist": true, "name": "/var/lib/kea/kea-leases4.csv" },
"valid-lifetime": 3600,
"subnet4": [
{
"id": 1,
"subnet": "10.239.4.0/24",
"pools": [ { "pool": "10.239.4.50 - 10.239.4.60" } ],
"option-data": [
{ "name": "routers", "data": "10.239.4.1" },
{ "name": "domain-name-servers", "data": "8.8.8.8" }
],
"reservations": [
{ "hw-address": "<MAC Laterano>", "ip-address": "10.239.4.15" }
]
}
]
}
}
```


Jalankan kea. Folder /run/kea hilang tiap restart, jadi wajib dibuat dulu:
```
mkdir -p /run/kea /var/lib/kea
kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
kea-dhcp4 -c /etc/kea/kea-dhcp4.conf &
```
Tunggu DHCP4_STARTED. Jendelanya jangan ditutup. Kalau butuh terminal lain, buka jendela tmux baru (Ctrl+b lalu c).




5. Victoria (DHCP subnet 3.0)
Sama seperti Yan, tetapi:
eth0 = 10.239.3.2/24, default via 10.239.3.1
config tanpa bagian reservations:
nano /etc/kea/kea-dhcp4.conf
```
"subnet": "10.239.3.0/24",
"pools": [ { "pool": "10.239.3.100 - 10.239.3.120" } ],
"routers" → "10.239.3.1"

{
"Dhcp4": {
"interfaces-config": { "interfaces": ["eth0"] },
"lease-database": { "type": "memfile", "persist": true, "name":
"/var/lib/kea/kea-leases4.csv" },
"valid-lifetime": 3600,
"subnet4": [
{
"id": 1,
"subnet": "10.239.3.0/24",
"pools": [ { "pool": "10.239.3.100 - 10.239.3.120" } ],
"option-data": [
{ "name": "routers", "data": "IP gateway Victoria" },
{ "name": "domain-name-servers", "data": "8.8.8.8" }
]
}
]
}
```
jalankan `mkdir -p /run/kea /var/lib/kea` lalu `kea-dhcp4 -c /etc/kea/kea-dhcp4.conf`, tunggu DHCP4_STARTED.
Gunakan kea-dhcp4, bukan udhcpd.




6. Laterano (web server, sender)
Ambil IP dari Yan:
```
udhcpc -i eth0
ip -br addr
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```
Harus 10.239.4.15/24. Kalau dapat IP lain, cek MAC di reservasi Yan.

Pastikan internet (gateway dan DNS dari DHCP), lalu install python3 kalau belum ada:
```
apk add python3
python3 --version
```

Buat script /root/myscripts/start_http.sh:
```
mkdir -p /root/myscripts && cd /root/myscripts
cat > start_http.sh << 'EOF'
#!/bin/sh
mkdir -p myweb
cd myweb
python3 -m http.server 9999 -d /root/myscripts/myweb
EOF
chmod +x start_http.sh
```

Buat file uji:
```
mkdir -p /root/myscripts/myweb && cd /root/myscripts/myweb
dd if=/dev/random of=tes1000.bin bs=1 count=1000
dd if=/dev/random of=tes10000.bin bs=10 count=1000
dd if=/dev/random of=tes50000.bin bs=50 count=1000
dd if=/dev/random of=tes5M.bin bs=5000 count=1000
ls -l
```

Ukuran harus tepat 1000, 10000, 50000, dan 5000000 byte.
5. Jalankan server:
```
tmux
cd /root/myscripts
./start_http.sh
```
Harus muncul Serving HTTP on 0.0.0.0 port 9999. Jendela ini jangan dipakai untuk perintah lain. Pakai jendela tmux baru untuk sysctl, iperf3, dan lain-lain.

Pastikan algoritma CUBIC (default):
```sysctl -w net.ipv4.tcp_congestion_control=cubic
ip tcp_metrics flush
sysctl net.ipv4.tcp_congestion_control
```
Jika `./start_http.sh` memberi Permission denied, jalankan `chmod +x start_http.sh`.

Cek algoritma: sysctl net.ipv4.tcp_congestion_control (default kondisi normal: cubic).
Untuk Bagian D, jalankan iperf3 -s di jendela terpisah dan hentikan setelah selesai. Kalau iperf3 belum terpasang, coba apk add iperf3.





7. Columbia (client)
Buka lewat VNC lalu xterm.
```
udhcpc -i eth0
ip -br addr
ip route
cat /etc/resolv.conf
ping -c 3 10.239.4.15
curl -I http://10.239.4.15:9999
```

Yang dicek:
- dapat IP 10.239.3.100 sampai .120
- ping sukses
- curl menampilkan size=10000B
Jebakan ketik yang sudah terjadi: % hilang di depan {, \n terketik /n, :9999 terketik .9999 atau :999, dan size_dowload (typo). Kalau output aneh, cek ulang perintahnya.
Firefox ke http://10.239.4.15:9999 boleh dipakai sebagai tes, tetapi menutupnya menghemat RAM.



8. Bagian E: Persiapan Kondisi Tanpa Gangguan
Di RhodesIsland (wajib, agar tidak ada sisa loss/delay dari bagian lain):
```
tc qdisc del dev eth1 root
tc qdisc del dev eth2 root
tc qdisc show dev eth1
tc qdisc show dev eth2
```
Hasil show harus hanya fq_codel (tanpa netem/tbf). Error Cannot delete qdisc with handle of zero aman. Cek ping -c 3 10.239.4.15 dari Columbia: RTT harus sekitar 1 ms.
Di Wireshark: Edit, Preferences, Protocols, TCP, matikan Relative sequence numbers (opsional; field Sequence Number (raw) selalu ada di detail paket).





9. Bagian E: Prosedur per Berkas
Ulangi untuk tes1000.bin, tes10000.bin, tes5M.bin, masing-masing satu koneksi bersih:
1. Klik kanan link Laterano-Hub1, Start capture, buka Wireshark, filter tcp.port == 9999.
2. Klik kanan link Columbia-Switch1, Start capture, filter yang sama.
3. Di xterm Columbia jalankan (ganti nama berkas):
```curl -o /dev/null -s -w "size=%{size_download}B speed=%{speed_download}B/s total=%{time_total}s\n" http://10.239.4.15:9999/tes1000.bin```
4. Screenshot output curl.
5. Stop kedua capture. Simpan: `E_tes1000_laterano.pcapng` dan `E_tes1000_columbia.pcapng` (dst.).
Pastikan capture dimulai sebelum curl agar paket SYN tertangkap. Satu capture berisi satu koneksi saja.



10. Bagian E: Cara Mengisi Tiap Baris Tabel
No. 1 - Ukuran berkas (B)
Dari output curl, angka setelah size=. Hasil: 1000, 10000, 5000000. Cocok dengan ls -l di Laterano.


No. 4 - Throughput (curl)
Angka setelah speed= (B/s) dari output curl yang sama.


No. 5 - Waktu total transfer (curl)
Angka setelah total= (detik) dari output curl yang sama.
No. 1, 4, 5 cukup dari satu screenshot curl per berkas.


No. 2 - Jumlah total segmen TCP
Buka pcap Laterano. Menu Statistics, Conversations, tab TCP. Pada baris koneksi 10.239.3.100 dan 10.239.4.15:9999, baca kolom Packets. Pastikan hanya ada satu baris koneksi (centang Limit to display filter jika perlu). Screenshot tabel Conversations.


No. 3 - RTT rata-rata (Laterano)
1. Pcap Laterano, filter ip.src == 10.239.3.100 && tcp.analysis.ack_rtt.
2. Klik satu paket, buka TCP, [SEQ/ACK analysis]. Baris The RTT to ACK the segment was berisi RTT.
3. Klik kanan baris itu, Apply as Column.
4. Untuk berkas besar: File, Export Packet Dissections, As CSV (Displayed), buka di Excel, =AVERAGE(kolom RTT). Untuk tes1000 yang paketnya sedikit, cukup jumlahkan beberapa nilai lalu bagi.
5. Nilai dalam detik, kalikan 1000 untuk ms. Screenshot kolom RTT dan hasil AVERAGE (beri label).


No. 6 - Sequence number awal client
Pcap Laterano, filter tcp.flags.syn==1 && tcp.flags.ack==0. Klik paket SYN dari 10.239.3.100, buka Transmission Control Protocol, catat Sequence Number (raw). Ini bilangan acak besar, bukan 0. Screenshot dengan filter terlihat.


No. 7 - Sequence number awal server
Filter tcp.flags.syn==1 && tcp.flags.ack==1. Klik paket SYN-ACK dari 10.239.4.15, catat Sequence Number (raw). Screenshot.


No. 8 - Jumlah ACK yang dikirim client
Pcap Laterano: Statistics, Endpoints, tab TCP (atau IPv4). Baca Packets Tx untuk alamat 10.239.3.100 (jumlah paket yang dikirim client dalam koneksi itu). Catatan: ini seluruh paket client, termasuk SYN, GET, dan FIN, bukan hanya ACK murni. Untuk hitungan ACK saja, pasang filter ip.src == 10.239.3.100 && tcp.flags.ack==1 dan baca Displayed di status bar, lalu tulis metode yang kamu pakai. Screenshot.


No. 9 - Status retransmisi (Laterano)
Filter tcp.analysis.retransmission. Baca Displayed di status bar; harapan 0 (daftar kosong). Screenshot dengan filter dan status bar terlihat.


No. 10 - Screenshot
Gabungan bukti: Wireshark dan output curl untuk tiap berkas. Letakkan di sel masing-masing.


Cara mengerjakan tabel
Aturan capture (berlaku untuk semua skenario)
- Start capture di kedua link dulu (Columbia ↔ Switch1 dan Laterano ↔ Hub1), filter tcp.port == 9999. Beri nama jelas (A4_normal_*, B_loss1_*, C_cubic_*, C_reno_*, E_1000_*, dan seterusnya).
- Baru jalankan curl. Kalau terbalik, handshake tidak tertangkap.
- Tunggu output curl, screenshot xterm, baru stop capture dan simpan.
- Cek frame 1 harus [SYN].
- Pisahkan file per skenario supaya data tidak tertukar.
- Pasang kolom dari panel detail dengan klik kanan, Apply as Column: Sequence Number (raw), Acknowledgment number (raw), RTT to ACK, TCP Segment Len.





A.4 (normal, tes10000)
No	Cara
1	Paket SYN: ip.src Columbia, ip.dst 10.239.4.15, port tujuan 9999
2	Tiga paket awal: SYN, SYN-ACK, ACK
3	Seq (raw) SYN, bandingkan dengan prediksi. ISN acak untuk mencegah spoofing dan bentrok dengan segmen lama
4	Seq (raw) SYN-ACK
5	Ack (raw) paket ACK = ISN server + 1
6	Filter ip.src == 10.239.4.15 && tcp.len > 0. Pilih segmen tengah, buktikan seq + len = seq berikutnya (contoh: 4549 + 1448 = 5997)
7	Filter ACK dari Columbia, tunjukkan nilai ack naik terus (kumulatif)
8	Filter tcp.analysis.ack_rtt, catat satu nilai RTT, jelaskan timer terlalu pendek (retransmisi palsu) dan terlalu panjang (tunggu lama)
9	Capture Columbia, tcp.analysis.retransmission (harusnya 0)
10	Capture Columbia, filter tcp.flags.fin == 1, tunjukkan FIN/ACK dan ACK

Nomor 1-8 pakai capture Laterano, 9-10 pakai Columbia.






E.1 (tiga ukuran, tanpa gangguan)
Hapus qdisc dulu (tc qdisc del dev eth1 root dan eth2), set Laterano ke cubic, lalu ulangi capture dan curl per file.

No	Cara
1, 4, 5	Output curl (size_download, speed_download, time_total)
2	Statistics → Conversations → tab TCP, kolom Packets (bukan tab Ethernet)
3	RTT: file kecil, rata-ratakan manual dari kolom RTT. File besar, I/O Graph AVG dengan interval besar
6, 7	Filter tcp.flags.syn == 1 dengan kolom Seq (raw): SYN = ISN client, SYN-ACK = ISN server
8	Statistics → Endpoints → tab TCP, Tx Packets milik IP Columbia
9	tcp.analysis.retransmission. Tulis apa adanya
10	Screenshot Wireshark dan curl





E.2: Jumlah segmen naik kira-kira proporsional dengan ukuran berkas, sedangkan RTT dan throughput tidak. Yang relatif tetap adalah pola handshake dan penutupan, serta ISN yang acak. time_total curl mencakup seluruh transaksi, sedangkan RTT hanya satu pasang segmen dan ACK.

Kesalahan yang sudah terjadi (hindari)
Capture dimulai setelah curl: handshake hilang, ulang.
Capture lama tersimpan dengan nama skenario lain: beri nama sebelum capture.
Lupa mkdir -p /run/kea: kea gagal start.
Route, NAT, dan resolv.conf hilang setelah restart: jalankan init.sh.
Menjalankan perintah di jendela tmux yang masih memegang server HTTP.
Memakai tab Ethernet untuk jumlah paket TCP.
Gangguan tc tertinggal saat skenario normal: selalu tc qdisc del dulu.
Hapus node berulang saat GNS3 lag: tunggu, klik sekali.

Kalau mau, panduan ini bisa kubuat jadi satu file (misalnya Markdown atau Word) supaya mudah disimpan dan dicetak.
