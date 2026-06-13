# Bölüm 5: Çekirdek Mimarisi ve Performans

WireGuard, performans söz konusu olduğunda rakipsizdir. Bu başarının arkasındaki sır, Linux çekirdeği (Kernel) ile olan derin entegrasyonu ve modern donanım mimarisini sonuna kadar kullanmasıdır.

## 1. Çekirdek Alanı (Kernel Space) Avantajı
OpenVPN gibi VPN'ler kullanıcı alanında (Userspace) çalışır. Bu durum, her paket için verinin çekirdekten kullanıcı alanına, sonra tekrar çekirdeğe kopyalanmasına (Context Switching) neden olur.
- **WireGuard**, doğrudan çekirdek içinde bir ağ arayüzü (`virtual network interface`) olarak çalışır.
- Veri kopyalama işlemleri minimize edilir (**Zero-copy** yaklaşımları).
- Bu sayede gecikme (latency) neredeyse yok seviyesine iner.

## 2. Çok Çekirdekli (Multi-core) Paralelleştirme
Modern işlemcilerde çok fazla çekirdek vardır. Klasik VPN'ler genelde tek bir işlem akışına (thread) takılı kalırken:
- WireGuard, paketleri çekirdeklere dağıtır.
- Şifreleme (Encryption) ve şifre çözme (Decryption) işlemleri eş zamanlı olarak tüm CPU çekirdeklerinde paralel olarak yapılır.
- **NAPI (New API)** desteği ile yüksek trafik altında CPU yükü dengeli dağıtılır.

## 3. GSO ve GRO Desteği
Ağ performansını artırmak için WireGuard şu teknikleri kullanır:
- **GSO (Generic Segmentation Offload)**: Büyük veri bloklarını tek bir büyük paketmiş gibi işleyip, donanım seviyesinde parçalara ayırır.
- **GRO (Generic Receive Offload)**: Gelen küçük paketleri birleştirerek işlemciye daha az kesme (interrupt) gönderilmesini sağlar.

Bu teknikler sayesinde 10Gbps ve üzeri hızlara ulaşmak WireGuard ile mümkündür.

## 4. Bellek Yönetimi (Memory Management)
WireGuard, çekirdek içinde sabit miktarda bellek kullanacak şekilde tasarlanmıştır.
- Dinamik bellek ayırma (dynamic allocation) işlemlerinden kaçınır. Bu, hem "Memory Leak" (bellek sızıntısı) riskini ortadan kaldırır hem de sistemin öngörülebilir olmasını sağlar.
- `sk_buff` (Linux socket buffer) yapısını çok verimli kullanarak paket işleme hattını (pipeline) optimize eder.

---
[Sonraki Bölüm: Güvenlik ve Stealth Mode >>](06_guvenlik_ve_stealth.md)
