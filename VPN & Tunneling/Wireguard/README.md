# WireGuard Master Class: Derinlemesine Teknik Analiz

Hoş geldin. Bu rehber, WireGuard VPN protokolünün minimalist yapısından çekirdek mekanizmalarına kadar uzanan bir teknoloji yolculuğudur. Akış, teoriden pratiğe ve oradan en alt katman olan sistem çekirdeğine doğru ilerler.

## İçerik Haritası

1.  **[Giriş ve Felsefe](01_giris_ve_felsefe.md)**: WireGuard'ın neden IPsec ve OpenVPN'e karşı bir devrim olduğunu, minimalist kod yapısını ve saldırı yüzeyi felsefesini inceler.
2.  **[Kriptografik Derin Dalış](02_kriptografi.md)**: Noise Protocol Framework, Curve25519, ChaCha20, Poly1305 ve BLAKE2s'in nasıl bir araya gelerek kırılamaz bir zincir oluşturduğunu açıklar.
3.  **[Protokol ve Paket Yapısı](03_protokol_ve_paketler.md)**: El sıkışma (Handshake) süreci, paket başlıkları ve veri iletiminin teknik detayları.
4.  **[CryptoKey Routing](04_cryptokey_routing.md)**: Public Key ile IP adreslerini birleştiren yönlendirme mantığı.
5.  **[Mimari ve Performans](05_mimari_performans.md)**: Linux çekirdek implementasyonu, multi-threading yetenekleri ve ağ yığını entegrasyonu.
6.  **[Güvenlik ve Stealth Mode](06_guvenlik_ve_stealth.md)**: DoS koruması, cookie mekanizmaları ve neden bir WireGuard sunucusunu "ping"leyemezsin.
7.  **[Kurulum Rehberi (Linux & Windows)](07_kurulum_linux_windows.md)**: Adım adım kurulum, anahtar üretimi ve ilk bağlantı testi.
8.  **[Pratik Uygulama ve Debug](08_pratik_ve_debug.md)**: İleri seviye yapılandırmalar ve sorun giderme teknikleri.
9.  **[Derin Teknik Analiz (Spesifikasyonlar)](09_derin_teknik_analiz.md)**: Bayt seviyesinde paket haritaları ve Linux Kernel kaynak kod analizi (RFC ve C kod seviyesi).
10. **[Çekirdek İçi Paket Yolculuğu](10_paket_yolculugu.md)**: Paketlerin kernel fonksiyonları arasındaki yolculuğu ve kuyruk yönetimi.

---
*Not: Bu dokümanlar profesyonel ağ mühendisleri, güvenlik araştırmacıları ve "her şeyin nasıl çalıştığını" merak edenler için hazırlanmıştır.*
