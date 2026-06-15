# Bölüm 12: WireGuard Performans Optimizasyonu (Tuning)

[<< Önceki Bölüm](11_dinamik_yonlendirme_bgp_ospf.md) | [Ana Sayfa](README.md)

---

WireGuard, tasarımındaki sıfır-kopyalama (zero-copy) yaklaşımı ve çoklu izlek (multithreading) yetenekleri sayesinde standart kurulumlarda bile olağanüstü hızlıdır. Ancak donanım sınırlarını (10Gbps, 40Gbps veya daha üstü) zorlamak ve veri merkezi ortamlarında maksimum verimi elde etmek için Linux çekirdeğinin ağ yığınına (Network Stack) müdahale edilmesi gerekir.

Bu bölümde, WireGuard tünellerinin performans darboğazlarını nasıl ortadan kaldıracağımızı inceleyeceğiz.

## 1. UDP Send ve Receive Arabelleklerini (Buffer) Artırmak

WireGuard verileri kapsülleyip UDP protokolü üzerinden taşıdığı için, yoğun trafik altında Linux çekirdeğinin standart UDP arabellekleri (buffer) dolabilir. Arabellek dolduğunda paket kayıpları (packet drop) başlar ve bu da performansı dramatik şekilde düşürür.

Bu durumu engellemek için `/etc/sysctl.conf` dosyasına şu parametreleri ekleyerek limitleri genişletmeliyiz:

```ini
# Maksimum UDP Receive (Alma) Buffer boyutu (Örn: 25 MB)
net.core.rmem_max=26214400
net.core.rmem_default=26214400

# Maksimum UDP Send (Gönderme) Buffer boyutu (Örn: 25 MB)
net.core.wmem_max=26214400
net.core.wmem_default=26214400

# Ağ kuyruğu (Network Queue) kapasitesini artırmak (Varsayılan 1000'dir)
net.core.netdev_max_backlog=50000
```

*Ayarları uygulamak için `sudo sysctl -p` komutunu çalıştırmanız yeterlidir.*

## 2. CPU Affinity ve NIC Kesmeleri (Interrupts)

WireGuard şifreleme işlemlerini çekirdeklere dağıtsa da, paketin fiziksel ağ kartından (NIC) çekirdeğe geliş sürecinde donanımsal kesmeler (hardware interrupts) oluşur.

10Gbps ve üzeri hızlarda, ağ kartının RX/TX (Alma/Gönderme) kuyruklarının belirli CPU çekirdeklerine bağlanması (Affinity) veya yükün tüm çekirdeklere eşit dağıtılması gerekir.

- Çoğu sistemde bu işlemi **`irqbalance`** servisi otomatik yapar (`sudo apt install irqbalance`).
- Ancak ekstrem senaryolarda `Receive-Side Scaling (RSS)` yapılandırması el ile (scriptler yardımıyla) işlemcilere bağlanarak donanım önbellek (cache) isabet oranları yükseltilir.

## 3. Rx/Tx Ring Buffer Boyutları

Fiziksel ağ kartınızın sürücü seviyesindeki kuyruk boyutunu (Ring Buffer) genişletmek, CPU'nun gelen paketleri işlemesine zaman kazandırır.
Mevcut ayarları `ethtool -g eth0` komutu ile görebilir, artırmak için aşağıdaki komutu kullanabilirsiniz:

```bash
sudo ethtool -G eth0 rx 4096 tx 4096
```

## 4. Tıkanıklık Kontrolü (BBR) ve FQ-CoDel

Eğer WireGuard tünelinden geçen TCP paketleri uzun mesafelerde (kıtalar arası veya yüksek ping'li bağlantılarda) hız kaybediyorsa, paket kayıplarına karşı daha dirençli olan Google'ın **BBR** algoritması kullanılmalıdır. Ayrıca bufferbloat sorununa karşı `fq_codel` algoritması önerilir:

```ini
# /etc/sysctl.conf içine ekleyin
net.core.default_qdisc=fq_codel
net.ipv4.tcp_congestion_control=bbr
```

Bu son optimizasyonlarla birlikte, WireGuard kurulumunuz modern internet omurgalarında veya ISP veri merkezlerinde hiçbir darboğaza takılmadan "hat hızı"nda (Line Rate) çalışacaktır.

---

[<< Önceki Bölüm](11_dinamik_yonlendirme_bgp_ospf.md) | [Ana Sayfa](README.md)
