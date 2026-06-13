# Bölüm 1: Giriş ve Minimalist Felsefe

WireGuard, Jason A. Donenfeld tarafından geliştirilen ve "basitlik her şeydir" felsefesini savunan modern bir VPN protokolüdür. Linus Torvalds'ın tabiriyle, "bir sanat eseri" olarak nitelendirilir. Peki neden?

## 1. Geçmişin Hantallığı: IPsec ve OpenVPN
Eski nesil VPN protokolleri (IPsec ve OpenVPN) muazzam bir karmaşıklığa sahiptir:
- **OpenVPN**: Yüz binlerce satır koddan oluşur, kullanıcı alanında (userspace) çalışır ve sertifika yönetimi oldukça zordur.
- **IPsec**: X.509 sertifikaları, IKEv2 el sıkışmaları ve onlarca farklı kriptografik seçenek (Cipher Suite) ile yönetilmesi bir kabustur.

Bu karmaşıklık, sadece konfigürasyon hatalarına değil, aynı zamanda devasa bir saldırı yüzeyine (Attack Surface) yol açar. Her bir satır kod, potansiyel bir güvenlik açığıdır.

## 2. WireGuard Felsefesi: "Security by Simplicity"
WireGuard'ın temel taşı **sadeliktir**. Yaklaşık **4.000 satır çekirdek (kernel) kodundan** oluşur. Karşılaştırma yaparsak:
- Linux Çekirdeği: ~30 Milyon Satır
- OpenVPN: ~100.000+ Satır
- WireGuard: ~4.000 Satır

### Avantajları:
1.  **Denetlenebilirlik (Auditability)**: Bir güvenlik araştırmacısı, WireGuard'ın tüm kaynak kodunu tek bir öğleden sonra oturup satır satır okuyabilir. Bu, IPsec veya OpenVPN için fiziksel olarak imkansızdır.
2.  **Performans**: Kullanıcı alanından (userspace) çekirdek alanına (kernel) bağlam geçişi (context switching) yapmaz. Bu, daha düşük gecikme (latency) ve daha yüksek bant genişliği demektir.
3.  **Hız**: Modern işlemcilerde ChaCha20 gibi primitifler sayesinde AES-NI donanım hızlandırmasına bile ihtiyaç duymadan inanılmaz bir hız sunar.

## 3. Kriptografik Çeviklik Yerine "Veledrom" Güvenliği
Geleneksel protokollerde "Crypto Agility" (kriptografik çeviklik) denen bir kavram vardır. Sunucu ve istemci hangi şifrelemeyi kullanacaklarını tartışırlar.
- **WireGuard bunu reddeder.**
- WireGuard, modern ve güvenli olduğu kanıtlanmış sabit bir "paket" sunar. Eğer yarın bu algoritmalardan biri kırılırsa, WireGuard versiyon atlar ve herkes yeni sürüme geçer. Bu, "downgrade attack" (sürüm düşürme saldırısı) riskini tamamen ortadan kaldırır.

---
[Sonraki Bölüm: Kriptografik Derin Dalış >>](02_kriptografi.md)
