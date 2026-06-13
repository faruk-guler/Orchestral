# Bölüm 2: Kriptografik Derin Dalış

WireGuard'ın derinlemesine teknik detaylarına inmek için onun kalbini, yani kriptografik primitiflerini ve **Noise Protocol Framework**'ü anlamalısın. WireGuard, bir eldiven gibi birbirine uyan seçkin bir algoritma seti kullanır.

## 1. Noise Protocol Framework (Noise_IK)
WireGuard, kendi el sıkışma protokolünü sıfırdan yazmak yerine modern bir standart olan **Noise Protocol Framework** üzerine inşa edilmiştir. Özel olarak **Noise_IK** desenini kullanır.

- **I (Identity)**: İstemci, sunucunun statik anahtarını bildiği varsayımıyla (pre-known) ilk mesajda kendi kimliğini (public key) gönderir.
- **K (Known)**: Sunucu, istemcinin kimliğini zaten listesinde (AllowedIPs) barındırıyorsa kabul eder.

Bu yapı sayesinde WireGuard **1-RTT (Round Trip Time)** hızında el sıkışır. Yani sadece karşılıklı birer paket gönderildiğinde bağlantı kurulmuş olur.

## 2. Kullanılan Algoritmalar (Primitifler)

### A. Curve25519 (Diffie-Hellman)
Anahtar değişimi için kullanılır. RSA veya klasik ECC (NIST eğrileri) yerine **Daniel J. Bernstein**'ın Curve25519 eğrisini tercih eder.
- **Neden?** Daha hızlıdır, sabit zamanda çalışır (side-channel saldırılarına dirençlidir) ve "backdoor" şüphesi taşımaz.

### B. ChaCha20 ve Poly1305 (AEAD)
Veri paketlerinin şifrelenmesi ve kimlik doğrulaması için kullanılır. 
- **ChaCha20**: Bir akış şifreleyicidir (stream cipher). AES'e göre ARM ve düşük güçlü işlemcilerde çok daha hızlıdır.
- **Poly1305**: Kimlik doğrulama kodu (MAC) üretir.
- **AEAD (Authenticated Encryption with Associated Data)**: Verinin hem gizli kalmasını hem de yolda değiştirilmediğini garanti eder.

### C. BLAKE2s (Hashing)
SHA-3'ten bile daha hızlı olan bu algoritma, anahtar karması (key hashing) ve tanımlayıcıların (indices) oluşturulmasında kullanılır.

### D. SipHash24
Ağ yığınındaki "hashtable" aramalarında kullanılır. Hash flooding DoS saldırılarına karşı koruma sağlar.

## 3. Anahtar Türetme ve Rotasyon (Rekeying)
WireGuard'da anahtarlar sonsuza kadar kullanılmaz.
- **Dinamik Rotasyon**: Her el sıkışmada (Handshake) yeni bir geçici (ephemeral) anahtar seti türetilir.
- **Perfect Forward Secrecy (PFS)**: Eğer birinin statik özel anahtarı (private key) çalınırsa, o anahtar çalınmadan önceki trafiği çözmesi mümkün değildir. Çünkü her oturumun anahtarı o an üretilen "ephemeral" anahtarlara bağlıdır.

## 4. Post-Quantum Preshared Key (PS)
WireGuard, kuantum bilgisayarların ECDH'yi çözme ihtimaline karşı opsiyonel bir koruma sunar. Konfigürasyona eklenecek bir **PresharedKey**, Noise handshake'ine karıştırılır. Bu sayede kuantum saldırılarına karşı simetrik bir güvenlik katmanı eklenir.

---
[Sonraki Bölüm: Protokol ve Paket Yapısı >>](03_protokol_ve_paketler.md)
