# Bölüm 4: CryptoKey Routing

WireGuard'ın en devrimci özelliklerinden biri **CryptoKey Routing** (Kripto-Anahtar Yönlendirme) mekanizmasıdır. Bu kavram, klasik VPN'lerdeki paket filtreleme ve yönlendirme karmaşasını tek bir basit eşleşmeye indirger.

## 1. Temel Kavram: Public Key = Kimlik = IP
Geleneksel VPN'lerde (örneğin IPsec), bir paketin hangi tünelden gideceğine karar veren karmaşık politikalar (policies) vardır. WireGuard'da ise her şey "Peer" (Eş) tabanlıdır.

Her Peer konfigürasyonunda iki kritik öğe bulunur:
1.  **Public Key**: Peer'ın kriptografik kimliği.
2.  **AllowedIPs**: O Peer'ın tünel içinde kullanmasına izin verilen IP adresleri.

## 2. Gönderme Süreci (Outbound)
WireGuard sanal arayüzüne (ör. `wg0`) bir paket geldiğinde çekirdek şunu yapar:
- Paketin **Hedef IP** (Destination IP) adresine bakar.
- Konfigürasyondaki tüm Peer'ların `AllowedIPs` listesini tarar.
- Hangi Peer'ın `AllowedIPs` listesinde bu hedef IP varsa, paketi o Peer'ın **Public Key**'i ile şifreler ve onun son bilinen (Endpoint) UDP adresine gönderir.

## 3. Alma Süreci (Inbound)
Bir UDP paketi `wg0` portuna geldiğinde:
- Paket çözülür (decryption). Paket başarıyla çözüldüyse, gönderen Peer'ın kimliği (Public Key) kesinleşmiş demektir.
- Çekirdek, şifresi çözülmüş iç paketin **Kaynak IP** (Source IP) adresine bakar.
- Eğer bu Kaynak IP, paketi gönderen Peer'ın `AllowedIPs` listesinde **yoksa**, paket anında çöpe atılır (drop).

### Neden Bu Çok Güvenli?
Bu mekanizma sayesinde **"IP Spoofing" (IP Sahteciliği)** WireGuard tüneli içinde imkansız hale gelir. Bir Peer, tünel içinden başka bir Peer'ın IP adresiyle paket gönderemez; çünkü çekirdek, o IP'nin o Public Key ile eşleşmediğini bilir.

## 4. Dinamik Endpoint Takibi (Roaming)
WireGuard Peer'ları sabit IP adreslerine sahip olmak zorunda değildir.
- Bir Peer (İstemci), IP adresi değişse bile (örneğin Wi-Fi'dan 4G'ye geçiş), sunucuya geçerli bir paket gönderdiği an sunucu onun yeni IP/Port kombinasyonunu (Endpoint) günceller.
- Bu özellik, mobil cihazlarda bağlantının hiç kopmamasını sağlar.

---
[Sonraki Bölüm: Çekirdek Mimarisi ve Performans >>](05_mimari_performans.md)
