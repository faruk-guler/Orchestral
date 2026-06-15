# Bölüm 9: Teknik Spesifikasyonlar ve Çekirdek (Kernel) Analizi

[<< Önceki Bölüm](08_pratik_ve_debug.md) | [Ana Sayfa](README.md) | [Sonraki Bölüm >>](10_paket_yolculugu.md)

---


Bu bölüm, genel anlatımın ötesinde, protokolün bayt seviyesindeki yapısını ve Linux çekirdeğindeki (drivers/net/wireguard/) kritik fonksiyonları inceler.

## 1. Paket Yapıları (Byte-Level Specification)

WireGuard paketleri sabit boyutlu ve **Little-Endian** hizasındadır.

### A. Handshake Initiation Packet (148 Bytes)
UDP payload'unun tam haritası:

| Offset | Boyut | İsim | Açıklama |
| :--- | :--- | :--- | :--- |
| 0 | 1 | `type` | Her zaman `0x01` (Handshake Initiation) |
| 1 | 3 | `reserved` | Sıfırlarla doldurulur (0x000000) |
| 4 | 4 | `sender_index` | İstemcinin rastgele oluşturduğu 32-bit ID |
| 8 | 32 | `unencrypted_ephemeral` | Curve25519 Ephemeral Public Key |
| 40 | 48 | `encrypted_static` | İstemcinin statik anahtarı (AEAD şifreli) |
| 88 | 28 | `encrypted_timestamp` | TAI64N formatında zaman damgası (Şifreli) |
| 116 | 16 | `mac1` | `BLAKE2s(Hash(Label + ResponderPublicKey) + Payload)` |
| 132 | 16 | `mac2` | DoS-Cookie (Eğer varsa) |

### B. Handshake Response Packet (92 Bytes)
UDP payload'unun tam haritası:

| Offset | Boyut | İsim | Açıklama |
| :--- | :--- | :--- | :--- |
| 0 | 1 | `type` | Her zaman `0x02` (Handshake Response) |
| 1 | 3 | `reserved` | Sıfırlarla doldurulur (0x000000) |
| 4 | 4 | `sender_index` | Sunucunun yerel 32-bit ID'si |
| 8 | 4 | `receiver_index` | İstemcinin gönderdiği `sender_index` değeri |
| 12 | 32 | `unencrypted_ephemeral` | Sunucunun geçici (ephemeral) public key'i |
| 44 | 16 | `encrypted_nothing` | Anahtar teyidi için boş veri şifrelemesi (AEAD) |
| 60 | 16 | `mac1` | `BLAKE2s(Hash(Label + InitiatorPublicKey) + Payload)` |
| 76 | 16 | `mac2` | DoS-Cookie doğrulaması (Eğer varsa) |

### C. Cookie Reply Packet (64 Bytes)
| Offset | Boyut | İsim | Açıklama |
| :--- | :--- | :--- | :--- |
| 0 | 1 | `type` | Her zaman `0x03` (Cookie Reply) |
| 1 | 3 | `reserved` | 0x000000 |
| 4 | 4 | `receiver_index` | Alıcının daha önce bildirdiği ID |
| 8 | 24 | `nonce` | Rastgele üretilen 24-byte nonce |
| 32 | 32 | `encrypted_cookie` | Şifrelenmiş cookie değeri |

### D. Transport Data Packet (Değişken Boyut)
| Offset | Boyut | İsim | Açıklama |
| :--- | :--- | :--- | :--- |
| 0 | 1 | `type` | Her zaman `0x04` (Data) |
| 1 | 3 | `reserved` | 0x000000 |
| 4 | 4 | `receiver_index` | Alıcının daha önce bildirdiği ID |
| 8 | 8 | `counter` | 64-bit Nonce (Replay attack koruması) |
| 16 | N | `payload` | Şifrelenmiş veri (ChaCha20-Poly1305) |

## 2. Linux Çekirdek Kod Analizi (`drivers/net/wireguard/`)

### 2.1. Paket Gönderimi: `wg_xmit()`
Bir paket tünel arayüzüne (`wg0`) geldiğinde çekirdek `device.c` içindeki `wg_xmit` fonksiyonunu çağırır.
```c
/* drivers/net/wireguard/device.c */
netdev_tx_t wg_xmit(struct sk_buff *skb, struct net_device *dev) {
    struct wg_device *wg = netdev_priv(dev);
    struct wg_peer *peer;

    // Hedef IP üzerinden Peer bulma (CryptoKey Routing)
    peer = wg_allowedips_lookup_dst(&wg->peer_allowedips, skb);
    if (unlikely(!peer)) goto err;

    // Paketi şifreleme kuyruğuna (workqueue) at
    wg_packet_encrypt_worker(&peer->packet_queue, skb);
}
```

### 2.2. Paralel Şifreleme: `wg_packet_encrypt_worker()`
WireGuard'ın hızlı olmasının sebebi şifreleme işini çekirdeklere dağıtmasıdır.
- `pad_packet()`: Paket boyutunu 16'nın katına tamamlar (Trafik analizi direnci).
- `chacha20poly1305_encrypt()`: Donanım hızlandırmalı (eğer varsa) AEAD işlemi başlar.

## 3. Noise Handshake Matematiksel Eyaletleri
El sıkışma sırasında `ck` (Chaining Key) ve `h` (Hash) değerleri sürekli güncellenir.

1.  **Başlangıç (Setup):**
    - `H = BLAKE2s("Noise_IK_25519_ChaChaPoly_BLAKE2s")`
2.  **MixHash:**
    - `H = BLAKE2s(H || data)`
3.  **MixKey (Rekeying):**
    - `(ck, k) = HKDF(ck, SharedSecret)`

Bu süreç, her mesajda yoldaki verinin bir özetini (transcript) tutarak, el sıkışmanın ortasında yapılacak bir değişikliği (Man-in-the-Middle) imkansız kılar.

[<< Önceki Bölüm](08_pratik_ve_debug.md) | [Ana Sayfa](README.md) | [Sonraki Bölüm >>](10_paket_yolculugu.md)
