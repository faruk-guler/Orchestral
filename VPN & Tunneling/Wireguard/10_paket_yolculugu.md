# Bölüm 10: Çekirdek İçi Paket Yolculuğu ve Kuyruk Yönetimi

[<< Önceki Bölüm: Derin Teknik Analiz](09_derin_teknik_analiz.md) | [Ana Sayfa >>](README.md)


Bu bölüm, WireGuard'ın Linux ağ yığını (Network Stack) içindeki tam yolculuğunu fonksiyon bazlı takip eder. Bu, protokolün en ince ayrıntısına kadar indiğimiz noktadır.

## 1. Gönderim Akışı (Transmission Path)
Bir paket tünel arayüzüne (`wg0`) girdiğinde sırasıyla şu duraklardan geçer:

1.  **`wg_xmit()`**: Paket karşılanır. Hedef IP'ye göre `AllowedIPs` araması yapılır ve ilgili `Peer` nesnesi bulunur.
2.  **`skb_queue_tail()`**: Paket, Peer'ın şifreleme kuyruğuna (encrypt queue) eklenir.
3.  **`wg_packet_encrypt_worker()`**: Bir `workqueue` (iş kuyruğu) tarafından tetiklenir. ChaCha20-Poly1305 şifrelemesi burada yapılır.
4.  **`wg_packet_send_staged_packets()`**: Şifrelenmiş paketler sıraya dizilir.
5.  **`udp_tunnel_xmit_skb()`**: WireGuard, paketi standart bir UDP paketi gibi internete (eth0 vb.) basar.

## 2. Alım Akışı (Receive Path)
Dışarıdan bir UDP paketi 51820 portuna geldiğinde:

1.  **`wg_receive()`**: UDP soketi paketi yakalar.
2.  **`noise_consume_initiation()`**: Eğer gelen bir el sıkışma paketi ise (Type 1), burada doğrulanır.
3.  **`wg_packet_decrypt_worker()`**: Eğer bir veri paketi ise (Type 4), şifresi çözülmek üzere işlemci çekirdeklerine dağıtılır.
4.  **`wg_packet_consume_data_done()`**: Şifre çözüldükten sonra "Replay Attack" kontrolü (counter check) yapılır.
5.  **`netif_receive_skb()`**: Şifresi çözülmüş ham paket, sanki fiziksel bir karttan gelmiş gibi Linux çekirdeğinin ana ağ işleme motoruna teslim edilir.

## 3. Kuyruk ve Çoklu İzlek (Multithreading) Mimarisi
WireGuard, CPU çekirdeklerini nasıl verimli kullanır?
- **Sıralı İşleme (Ordering)**: Şifreleme paralel yapılsa bile, paketlerin tünelden orijinal sırasıyla çıkması zorunludur. WireGuard bunun için paketlere bir `sequence number` ekler ve çıkışta bunları sıraya dizer.
- **Yük Dengeleme**: Her işlemci çekirdeği için ayrı bir `workqueue` oluşturulur. Bu da tek bir CPU'nun darboğaz (bottleneck) olmasını engeller.

## 4. Akış Özeti (Function Trace)
```text
[Uygulama] -> [wg0] -> wg_xmit() -> wg_packet_encrypt_worker() -> [UDP/eth0] -> [İnternet]
                                                                        |
                                                                        v
[Uygulama] <- [wg0] <- netif_receive_skb() <- wg_packet_decrypt_worker() <- [UDP/eth0]
```

Bu döngü, WireGuard'ın neden 10Gbps hızlarında bile düşük CPU kullanımıyla çalışabildiğinin cevabıdır.
