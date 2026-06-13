# Bölüm 3: Protokol Akışı ve Paket Yapısı

WireGuard, UDP üzerinden çalışır ve sadece **4 ana paket türü** vardır. Bu basitlik, protokolün hızı ve güvenliğinin anahtarıdır.

## 1. Paket Türleri ve Yapıları

Tüm WireGuard paketleri küçük bir başlık (header) ile başlar. İlk byte her zaman paket tipini belirler:

### A. Handshake Initiation (Tip: 1)
İstemcinin (Initiator) el sıkışmayı başlatmak için gönderdiği pakettir.
- **Sender Index**: İstemcinin yerel olarak oluşturduğu benzersiz bir ID (32-bit). Sunucu, istemciye cevap verirken bu ID'yi kullanır.
- **Unencrypted Ephemeral**: İstemcinin o anlık oluşturduğu geçici public key.
- **Encrypted Static**: İstemcinin asıl public key'i (Şifrelenmiş olarak gönderilir, gizlilik sağlar).
- **MAC1/MAC2**: DoS koruması için kullanılan doğrulamalar (Bkz. Bölüm 6).

### B. Handshake Response (Tip: 2)
Sunucunun (Responder) cevabıdır.
- **Sender Index**: Sunucunun yerel ID'si.
- **Receiver Index**: İstemcinin ilk pakette gönderdiği Sender Index.
- **Ephemeral**: Sunucunun geçici public key'i.
- **MAC1/MAC2**: Sunucuyu koruyan doğrulamalar.

### C. Cookie Reply (Tip: 3)
Eğer sunucu ağır bir yük (Load) altındaysa ve DoS saldırısı şüphesi varsa, el sıkışma paketini reddeder ve istemciye bir "Cookie" gönderir.

### D. Transport Data (Tip: 4)
Asıl verinin (VPN trafiği) taşındığı pakettir.
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

---
[Sonraki Bölüm: CryptoKey Routing >>](04_cryptokey_routing.md)
