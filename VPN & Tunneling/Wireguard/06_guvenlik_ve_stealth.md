# Bölüm 6: Güvenlik, Stealth Mode ve DoS Koruması

WireGuard, sadece şifreleme ile değil, aynı zamanda tünelin dış dünyadan nasıl göründüğü (veya görünmediği) ile de ilgilenir. Bu bölümde, "sessizlik" felsefesini ve DoS (Denial of Service) saldırılarına karşı savunma mekanizmalarını inceleyeceğiz.

## 1. Stealth Mode: Görünmezlik
WireGuard tüneli açık olsa bile, bir saldırgan sunucunuzun UDP portunu taradığında (nmap vb.) portun "kapalı" veya "filtrelenmiş" olduğunu sanacaktır.
- **Neden?** WireGuard, gelen paketlerin kimliğini doğrulamadan asla yanıt vermez.
- Eğer paket geçerli bir MAC1 (Bkz. aşağıda) içermiyorsa, WireGuard sessiz kalır (Silently drop). Yanıt bile dönmez (ICMP Unreachable vb. yok).
- Bu da internet üzerindeki rastgele tarayıcılardan (masscan, shodan) gizlenmeyi sağlar.

## 2. DoS Koruması ve Cookie Mekanizması
El sıkışma (Handshake) paketleri işlemci için maliyetlidir (Curve25519 hesaplaması). Bir saldırgan milyonlarca sahte handshake paketi göndererek sunucuyu yorabilir. WireGuard buna karşı iki katmanlı bir MAC (Message Authentication Code) sistemi kullanır:

### MAC1 (Kimlik Doğrulama)
Her mesajda bulunur. Gönderen tarafın (Peer) public key'i üzerinden hesaplanır. Eğer bu MAC1 yanlışsa, paket daha CPU'yu yormadan hemen çöpe atılır.

### MAC2 (Cookie Mekanizması)
Eğer sunucu yoğun yük altındaysa (Load threshold aşılmışsa):
1. Sunucu paketlere cevap vermeyi keser.
2. Bunun yerine istemciye bir **Cookie Reply** gönderir.
3. İstemci, bir sonraki el sıkışma paketine bu "Cookie"yi ve IP adresini içeren bir **MAC2** eklemek zorundadır.
4. Bu, sunucunun IP spoofing (IP sahteciliği) yapan saldırganları ayırt etmesini sağlar (çünkü saldırgan o IP'ye giden Cookie'yi göremez).

## 3. Bilgi Sızıntısını Önleme (Identity Hiding)
WireGuard paketleri içinde Peer'ların isimleri, e-postaları veya diğer tanımlayıcı bilgileri asla açık metin olarak geçmez.
- İstemcinin statik anahtarı (Static Key), ilk el sıkışma paketinde geçici (ephemeral) anahtar ile şifrelenir.
- Bir ağ dinleyicisi (Sniffer), paketin kime ait olduğunu teknik olarak anlayamaz.

## 4. Key Rotation (Anahtar Yenileme)
Sürekli veri akışı olsa dahi, her tünelin anahtarı belirli aralıklarla (yaklaşık 2 dakikada bir) yenilenir. Bu, verinin bir kısmı çalınsa bile diğer kısımlarının güvenliğini korur (Perfect Forward Secrecy).

## 5. NAT Traversal ve PersistentKeepalive
WireGuard bir "sessiz" protokoldür; veri akışı yoksa paket göndermez. Ancak çoğu ev yönlendiricisi (NAT), 30-120 saniye boyunca işlem görmeyen UDP oturumlarını kapatır.
- **Problem**: Oturum kapanırsa, sunucudan istemciye gelen paketler yönlendirici tarafından reddedilir.
- **Çözüm**: `PersistentKeepalive = 25` ayarı.
- **Neden 25 Saniye?**: Endüstri standardı NAT zaman aşımlarının alt sınırı olan 30 saniyenin hemen altındadır. Her 25 saniyede bir gönderilen küçük, şifreli ve kimliği doğrulanmış "boş" bir paket, NAT tablosundaki girdinin taze kalmasını sağlar.

---
[Sonraki Bölüm: Kurulum Rehberi (Linux & Windows) >>](07_kurulum_linux_windows.md)
