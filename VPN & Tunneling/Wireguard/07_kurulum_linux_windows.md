# Bölüm 7: Linux ve Windows Kurulum Rehberi

[<< Önceki Bölüm](06_guvenlik_ve_stealth.md) | [Ana Sayfa](README.md) | [Sonraki Bölüm >>](08_pratik_ve_debug.md)

---

Bu bölümde, her iki platformda WireGuard'ı sıfırdan ayağa kaldırmayı ve ilk başarılı tüneli kurmayı öğreneceğiz.

## 1. Linux Kurulumu (KMT: Komut Satırı)

Linux'ta WireGuard, 5.6 sürümünden beri çekirdeğe (kernel) dahildir. Eski sürümlerde modül olarak yüklenir.

### A. Ubuntu / Debian
```bash
sudo apt update
sudo apt install wireguard -y
```

### B. CentOS / RHEL (EPEL Gerektirir)
```bash
sudo yum install epel-release elrepo-release -y
sudo yum install kmod-wireguard wireguard-tools -y
```

### C. Anahtar Üretimi (Key Generation)
Sunucu ve istemci için anahtarlar aynı komutla üretilir:
```bash
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
```
*Not: `umask 077` komutu, üretilen dosyaların sadece sahibi tarafından okunabilmesini sağlar.*

### D. Konfigürasyon Dosyasının Oluşturulması
Servisi başlatmadan önce `/etc/wireguard/wg0.conf` dosyasını oluşturup gerekli tanımlamaları yapmalısınız. (Konfigürasyon dosyasının ayrıntılı yapısı için bkz. [Bölüm 8: İleri Seviye Konfigürasyon](08_pratik_ve_debug.md)).

### E. Servisi Başlatma
```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

---

## 2. Windows Kurulumu (GUI)

Windows tarafında WireGuard, kullanıcı dostu bir arayüz ve sistem tepsisi (system tray) uygulaması sunar.

1.  **İndirme**: [wireguard.com/install](https://www.wireguard.com/install/) adresinden `.msi` yükleyicisini indir ve kur.
2.  **Boş Tünel Ekle**: Uygulamayı aç, "Add Tunnel" yanındaki oka tıkla ve "Add empty tunnel..." seç.
3.  **Otomatik Anahtar**: Windows istemcisi senin için otomatik bir PrivateKey ve PublicKey çifti üretir.
4.  **Konfigürasyon**:
    ```ini
    [Interface]
    PrivateKey = <Senin_Windows_Özel_Anahtarın>
    Address = 10.0.0.2/32
    DNS = 1.1.1.1

    [Peer]
    PublicKey = <Sunucunun_Public_Keyi>
    Endpoint = <Sunucu_IP_Adresi>:51820
    AllowedIPs = 0.0.0.0/0
    ```
5.  **Aktifleştir**: "Activate" butonuna bas.

---

## 3. İlk Test: "Handshake" Doğrulaması

Kurulumun başarılı olduğunu anlamanın tek yolu sunucu tarafında şu komutu vermektir:
```bash
sudo wg show
```
Eğer çıktı şu şekildeyse bağlantı tamamdır:
`latest handshake: 5 seconds ago`
`transfer: 1.2 KiB received, 800 B sent`

---

## 4. IPv6 Desteği (Dual-Stack)
WireGuard doğuştan çift yığınlıdır (Dual-stack). Sunucu ve istemci arasında hem IPv4 hem de IPv6 tüneli kurmak için:
- **Interface/Address**: Hem IPv4 hem IPv6 adresi verin (Örn: `10.0.0.1/24, fd00::1/64`).
- **AllowedIPs**: Her iki protokolün bloklarını ekleyin (Örn: `0.0.0.0/0, ::/0`).

---

## 5. Mobil İstemciler (Android & iOS) ve QR Kod ile Kurulum
Mobil cihazlarda konfigürasyonu elle girmek yerine bir QR kodunu taratarak saniyeler içinde bağlantıyı kurabilirsiniz.

1. **Linux Sunucuda qrencode Kurulumu:**
   ```bash
   sudo apt install qrencode -y
   ```
2. **QR Kod Üretimi:**
   İstemci konfigürasyon dosyasını (`client.conf`) terminale QR kod olarak yazdırmak için:
   ```bash
   qrencode -t ansiutf8 < client.conf
   ```
   Bu komut terminal ekranında büyük bir QR kod oluşturacaktır. Mobil cihazınızdaki resmi WireGuard uygulamasını açıp **Create from QR code** (QR Koddan Oluştur) seçeneğiyle bu kodu kameraya taratmanız yeterlidir.

---
[<< Önceki Bölüm](06_guvenlik_ve_stealth.md) | [Ana Sayfa](README.md) | [Sonraki Bölüm >>](08_pratik_ve_debug.md)
