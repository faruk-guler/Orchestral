# Bölüm 9: Teknik Spesifikasyonlar ve Çekirdek (Kernel) Analizi

[<< Önceki Bölüm: Pratik ve Debug](08_pratik_ve_debug.md) | [Sonraki Bölüm: Paket Yolculuğu >>](10_paket_yolculugu.md)


Bu bölüm, genel anlatımın ötesinde, protokolün bayt seviyesindeki yapısını ve Linux çekirdeğindeki (drivers/net/wireguard/) kritik fonksiyonları inceler.

## 1. Paket Yapıları (Byte-Level Specification)

WireGuard paketleri sabit boyutlu ve **Little-Endian** hizzasındadır.

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

### B. Transport Data Packet (Değişken Boyut)
| Offset | Boyut | İsim | Açıklama |
| :--- | :--- | :--- | :--- |
| 0 | 1 | `type` | Her zaman `0x04` (Data) |
| 1 | 3 | `reserved` | 0x000000 |
| 4 | 4 | `receiver_index` | Alıcının daha önce bildirdiği ID |
| 8 | 8 | `counter` | 64-bit Nonce (Replay attack koruması) |
| 16 | N | `payload` | Şifrelenmiş veri (ChaCha20-Poly1305) |

## 2. Linux Çekirdek Kod Analizi (`drivers/net/wireguard/`)

### 2.1. Paket Gönderimi: `wg_xmit()`
Bir paket tünel arayüzüne (`wg0`) geldiğinde çekirdek `send.c` içindeki `wg_xmit` fonksiyonunu çağırır.
```c
/* drivers/net/wireguard/send.c */
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

---
[Sonraki Bölüm: Çekirdek İçi Paket Yolculuğu >>](10_paket_yolculugu.md)
