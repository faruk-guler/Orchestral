# Bölüm 8: İleri Seviye Konfigürasyon ve Hata Ayıklama

[<< Önceki Bölüm: Kurulum](07_kurulum_linux_windows.md) | [Sonraki Bölüm: Derin Teknik Analiz >>](09_derin_teknik_analiz.md)

Kuramsal bilgileri pratiğe dökme zamanı. WireGuard, basit bir `INI` formatında konfigürasyon dosyası kullanır (`/etc/wireguard/wg0.conf`).

## 1. Konfigürasyon Yapısı (Anatomi)

```ini
[Interface]
PrivateKey = <Sunucu_Özel_Anahtarı>
Address = 10.0.0.1/24
ListenPort = 51820
# IP Yönlendirme ve Güvenlik Duvarı Kuralları
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <İstemci_Genel_Anahtarı>
AllowedIPs = 10.0.0.2/32
PresharedKey = <Opsiyonel_Kuantum_Koruması>
```

- **PostUp/PostDown**: Tünel açıldığında ve kapandığında çalışan betiklerdir. Genellikle NAT ve yönlendirme (forwarding) için kullanılır.
- **AllowedIPs**: Sunucu tarafında "bu istemci hangi iç IP'leri kullanabilir?" sorusuna yanıt verir.

## 2. MTU ve MSS Clamping (En Büyük Sorun)
VPN tünelleri, paketin üzerine kendi başlıklarını (header) ekler. Bu durum, paketin orijinal MTU (genellikle 1500) boyutunu aşmasına neden olabilir.
- **Semptom**: Web siteleri yavaş açılır veya SSH bağlantıları "donar".
- **Çözüm**: MTU değerini manuel olarak 1420 veya 1280'e düşürmek (WireGuard başlığı 80 byte kaplar).
- **MSS Clamping**: `iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu` komutu ile TCP paketlerinin MSS değerini otomatik ayarlamak en sağlıklı yoldur.

## 3. Hata Ayıklama (Troubleshooting)

### A. Tünel Durumu
`wg show` komutu ile gerçek zamanlı istatistikleri gör:
- `latest handshake`: Eğer bu kısım boşsa veya 3-4 dakikadan fazlaysa, taraflar el sıkışamamıştır (Genelde Firewall veya yanlış Public Key).
- `transfer`: Veri gidip gelip gelmediğini kontrol et.

### B. Paket Analizi
`tcpdump -i wg0` veya `tcpdump -i any udp port 51820` komutları ile paketlerin tünele girip girmediğini veya şifrelenmiş paketlerin dışarı çıkıp çıkmadığını izleyebilirsin.

### C. Çekirdek Logları
Eğer çekirdek seviyesinde bir sorun varsa:
`dmesg | tail` veya `journalctl -k` komutları WireGuard modülünün hata mesajlarını gösterir.

4.  **IP Forwarding**: Linux sunucuda `sysctl -w net.ipv4.ip_forward=1` yapılmamışsa paketler tünelden dışarı (İnternete) çıkamaz.

## 5. DNS Güvenliği ve Kaçak (Leak) Önleme
VPN'lerde en sık karşılaşılan sorun DNS sorgularının tünel dışına taşmasıdır.
- **DNS Parametresi**: `[Interface]` altına `DNS = 1.1.1.1` eklemek, sistemin tüm DNS sorgularını tünel içine zorlamasını sağlar.
- **IPv6 Sızıntısı**: Eğer sunucu IPv6 desteklemiyorsa, `AllowedIPs` kısmına `::/0` eklememek Windows'un native IPv6 üzerinden DNS sızdırmasına neden olabilir. Her zaman `AllowedIPs = 0.0.0.0/0, ::/0` kullanarak tüm yolları kapatın.

---
[Sonraki Bölüm: Derin Teknik Analiz >>](09_derin_teknik_analiz.md)
