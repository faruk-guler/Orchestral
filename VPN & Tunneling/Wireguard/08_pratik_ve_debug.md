# Bölüm 8: İleri Seviye Konfigürasyon ve Hata Ayıklama

[<< Önceki Bölüm](07_kurulum_linux_windows.md) | [Ana Sayfa](README.md) | [Sonraki Bölüm >>](09_derin_teknik_analiz.md)

---

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
  *Önemli Not: `PostUp` ve `PostDown` komutlarındaki `eth0` parametresini sunucunuzun internete bağlı olan gerçek dış ağ arayüzü adı (ör. `eth0`, `ens3`, `enp3s0` veya `wlan0`) ile değiştirmelisiniz. Aksi takdirde istemciler internete erişemez.*
- **AllowedIPs**: Sunucu tarafında "bu istemci hangi iç IP'leri kullanabilir?" sorusuna yanıt verir.
- **Table = off**: (Opsiyonel) `wg-quick` aracının varsayılan yönlendirme tablolarını (routing tables) otomatik oluşturmasını engeller. Kendi özel (custom) yönlendirme kurallarınızı yazacağınız karmaşık ağ topolojilerinde kritik bir ayardır.

## 2. MTU ve MSS Clamping (En Büyük Sorun)
VPN tünelleri, paketin üzerine kendi başlıklarını (header) ekler. Bu durum, paketin orijinal MTU (genellikle 1500) boyutunu aşmasına neden olabilir.
- **Semptom**: Web siteleri yavaş açılır veya SSH bağlantıları "donar".
- **Tünel Ek Yükü (Overhead) Hesabı**:
  - **IPv4 için**: IP Başlığı (20B) + UDP Başlığı (8B) + WireGuard Veri Başlığı (16B) + Poly1305 Doğrulama Etiketi (16B) = **60 Bayt**.
  - **IPv6 için**: IPv6 Başlığı (40B) + UDP Başlığı (8B) + WireGuard Veri Başlığı (16B) + Poly1305 Doğrulama Etiketi (16B) = **80 Bayt**.
- **Çözüm**: Tünelin hem IPv4 hem de IPv6 paketlerini fragmente etmeden taşıyabilmesi için en güvenli senaryo olan IPv6 ek yükü (80 bayt) baz alınır. Standart 1500 MTU'lu bir hatta WireGuard MTU'su **1420** (1500 - 80) olarak ayarlanmalıdır. PPPoE gibi 1492 MTU kullanan ADSL/Fiber hatlarda ise bu değer **1412**'ye düşürülmelidir. Güvenli bir alt sınır olarak **1280** (IPv6'nın izin verdiği minimum MTU) da tercih edilebilir.
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

## 4. IP Yönlendirme (IP Forwarding)
Linux sunucuda IPv4 için `sysctl -w net.ipv4.ip_forward=1` ve IPv6 için `sysctl -w net.ipv6.conf.all.forwarding=1` yapılmamışsa paketler tünelden dışarı (İnternete) çıkamaz.

## 5. DNS Güvenliği ve Kaçak (Leak) Önleme
VPN'lerde en sık karşılaşılan sorun DNS sorgularının tünel dışına taşmasıdır.
- **DNS Parametresi**: `[Interface]` altına `DNS = 1.1.1.1` eklemek, sistemin tüm DNS sorgularını tünel içine zorlamasını sağlar.
- **IPv6 Sızıntısı**: Eğer sunucu IPv6 desteklemiyorsa, `AllowedIPs` kısmına `::/0` eklememek Windows'un native IPv6 üzerinden DNS sızdırmasına neden olabilir. Her zaman `AllowedIPs = 0.0.0.0/0, ::/0` kullanarak tüm yolları kapatın.

## 6. Docker ve Konteyner Ortamı (Gotchas)
WireGuard, çekirdek (kernel) seviyesinde çalıştığı için standart bir Docker konteynerinde doğrudan çalışamaz. İki seçeneğiniz vardır:
1. Konteyneri `--cap-add=NET_ADMIN` yetkisiyle ve `--sysctl net.ipv4.ip_forward=1` parametresiyle başlatmak (Ayrıca host işletim sisteminde `wireguard` modülünün yüklü olması gerekir).
2. Eğer host sistemde kernel erişiminiz yoksa, konteyner içinde `wireguard-go` veya `BoringTun` gibi kullanıcı alanı (userspace) alternatiflerini kullanmak.

[<< Önceki Bölüm](07_kurulum_linux_windows.md) | [Ana Sayfa](README.md) | [Sonraki Bölüm >>](09_derin_teknik_analiz.md)
