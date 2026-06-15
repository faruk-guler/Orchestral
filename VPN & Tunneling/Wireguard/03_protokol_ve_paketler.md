# Bölüm 3: Protokol Akışı ve Paket Yapısı

[<< Önceki Bölüm](02_kriptografi.md) | [Ana Sayfa](README.md) | [Sonraki Bölüm >>](04_cryptokey_routing.md)

---

WireGuard, UDP üzerinden çalışır ve sadece **4 ana paket türü** vardır. Bu basitlik, protokolün hızı ve güvenliğinin anahtarıdır.

## 1. Paket Türleri ve Yapıları

Tüm WireGuard paketleri küçük bir başlık (header) ile başlar. İlk byte her zaman paket tipini belirler:

### A. Handshake Initiation (Tip: 1)
İstemcinin (Initiator) el sıkışmayı başlatmak için gönderdiği, sabit **148 bayt** boyutundaki pakettir. (Detaylı bayt haritası için bkz. Bölüm 9)
- **Sender Index**: İstemcinin yerel olarak oluşturduğu benzersiz bir ID (32-bit). Sunucu, istemciye cevap verirken bu ID'yi kullanır.
- **Unencrypted Ephemeral**: İstemcinin o anlık oluşturduğu geçici public key.
- **Encrypted Static**: İstemcinin asıl public key'i (Şifrelenmiş olarak gönderilir, gizlilik sağlar).
- **MAC1/MAC2**: DoS koruması için kullanılan doğrulamalar (Bkz. Bölüm 6).

### B. Handshake Response (Tip: 2)
Sunucunun (Responder) cevabıdır ve sabit **92 bayt** boyutundadır.
- **Sender Index**: Sunucunun yerel ID'si (32-bit).
- **Receiver Index**: İstemcinin ilk pakette gönderdiği Sender Index.
- **Ephemeral**: Sunucunun geçici public key'i.
- **Encrypted Nothing**: Anahtar teyidi sağlamak için şifrelenmiş boş veri alanı (16 baytlık AEAD doğrulaması).
- **MAC1/MAC2**: Sunucuyu koruyan doğrulamalar.

### C. Cookie Reply (Tip: 3)
Eğer sunucu ağır bir yük (Load) altındaysa ve DoS saldırısı şüphesi varsa, el sıkışma paketini reddeder ve istemciye sabit **64 bayt** boyutunda bir "Cookie Reply" paketi gönderir.
- **Receiver Index**: Alıcının daha önceden belirlediği ID.
- **Nonce**: Rastgele üretilen 24 baytlık değer (XChaCha20Poly1305 nonce).
- **Encrypted Cookie**: Şifrelenmiş cookie değeri.

### D. Transport Data (Tip: 4)
Asıl verinin (VPN trafiği) taşındığı değişken boyutlu pakettir.
- **Receiver Index**: Alıcının daha önceden belirlediği ID.
- **Counter**: 64-bitlik bir sayaç. Replay attack (tekrar saldırısı) koruması sağlar.
- **Encrypted Payload**: Şifrelenmiş veri.

## 2. El Sıkışma Süreci (Handshake)
WireGuard'da bağlantı kurmak saniyeler değil, milisaniyeler sürer:

1.  **Initiator -> Responder**: "Merhaba, benim ID'im A, geçici anahtarım X, statik kimliğim şifreli olarak ekte."
2.  **Responder -> Initiator**: "Merhaba A, benim ID'im B, senin ID'n A olduğunu anladım, benim geçici anahtarım Y."
3.  **Bitti**: Artık her iki taraf da `X` ve `Y` üzerinden ortak bir gizli anahtar (Shared Secret) türetmiş durumdadır.

## 3. Zamanlayıcılar (Timers) ve Durum Yönetimi
WireGuard'ın "stateless" (durumsuz) gibi görünmesinin sebebi akıllı zamanlayıcılardır:
- **REKEY_AFTER_TIME**: Bir anahtar çifti 2 dakikadan fazla kullanılmaz.
- **REJECT_AFTER_TIME**: Bir anahtar çifti 3 dakikadan sonra geçersiz sayılır.
- **KEEPALIVE**: Eğer trafik yoksa, tünelin açık kalması için (NAT arkasındaki cihazlar için kritik) her 25 saniyede bir boş paket gönderilebilir.

[<< Önceki Bölüm](02_kriptografi.md) | [Ana Sayfa](README.md) | [Sonraki Bölüm >>](04_cryptokey_routing.md)
