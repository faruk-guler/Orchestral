# Bölüm 11: WireGuard Üzerinde Dinamik Yönlendirme (BGP ve OSPF)

[<< Önceki Bölüm](10_paket_yolculugu.md) | [Ana Sayfa](README.md) | [Sonraki Bölüm >>](12_performans_optimizasyonu.md)

---

WireGuard'ın merkezinde yer alan **CryptoKey Routing** mantığı (IP adreslerini Public Key'ler ile eşleştiren "AllowedIPs" tablosu) minimalist ve son derece güvenli bir yapı sunar. Ancak büyük ölçekli şirket (Enterprise) ağlarında, çoklu bulut (Multi-Cloud) ve onlarca şubenin (Site-to-Site) birbirine bağlandığı senaryolarda bu durum bir dezavantaja dönüşebilir.

Her yeni alt ağ (subnet) eklendiğinde tüm uçlardaki (endpoints) `AllowedIPs` listesini manuel güncellemek ölçeklenebilir değildir. İşte bu noktada **Dinamik Yönlendirme (Dynamic Routing)** devreye girer.

## 1. Neden BGP? Neden OSPF Değil?

Dinamik yönlendirmede iki endüstri standardı vardır: OSPF ve BGP.

- **OSPF (Open Shortest Path First):** Yeni komşuları bulmak için Layer 2 (Ethernet) üzerinden *Multicast* yayınları yapar. Ancak **WireGuard bir Layer 3 (IP) tünelidir** ve doğası gereği Multicast (çoklu yayın) ve Broadcast paketlerini taşımaz. OSPF'i WireGuard üzerinden çalıştırmak için GRE gibi ek tünelleme katmanlarına ihtiyaç duyarsınız (bu da performansı öldürür ve MTU sorunları yaratır).
- **BGP (Border Gateway Protocol):** Komşularla iletişim kurmak için saf *Unicast TCP (Port 179)* kullanır. WireGuard'ın çalışma mantığına **kusursuz** uyum sağlar. İnternetin omurgasını oluşturan BGP, günümüzde WireGuard kurumsal ağlarında da de facto standarttır.

## 2. Kilit Mimari Çözüm: "Noktadan Noktaya" (Point-to-Point) Arayüzler

BGP'nin çalışması için, karşıdan hangi hedefe ait paket gelirse gelsin WireGuard'ın o paketi kabul etmesi ve Linux çekirdeğine iletmesi gerekir. Bunun tek yolu `AllowedIPs = 0.0.0.0/0` yapmaktır.

Ancak **Bölüm 4**'ten hatırlayacağınız üzere: WireGuard'da aynı arayüz (örneğin `wg0`) içindeki birden fazla Peer'a aynı `AllowedIPs` değerini veremezsiniz! Verirseniz trafik çakışır ve en son eklenen Peer IP'nin sahibi olur.

### Çözüm Mimarisinin Altın Kuralı

BGP veya OSPF çalıştıracaksanız, **her bir komşu (şube) için ayrı bir sanal ağ arayüzü (Interface)** oluşturmalısınız!

- Merkez (HQ) -> Şube A için `wg1` arayüzü (AllowedIPs = 0.0.0.0/0)
- Merkez (HQ) -> Şube B için `wg2` arayüzü (AllowedIPs = 0.0.0.0/0)

Bu mimariye **Point-to-Point (P2P)** tünelleme denir. Bu sayede her bir tünel bağımsız bir kablo gibi davranır ve yönlendirme kararları WireGuard tarafından değil, Linux çekirdeği (Kernel Routing Table) ve BGP yazılımı tarafından verilir.

## 3. `Table = off` Parametresinin Önemi

Eğer her komşu için ayrı bir wg arayüzü açar ve `AllowedIPs = 0.0.0.0/0` yaparsanız, `wg-quick` aracı varsayılan olarak sunucunun tüm internet trafiğini bu tünellere göndermeye çalışır ve sunucunuzun interneti kopar.

Bunu engellemek için WireGuard konfigürasyonunuza şu satırı eklemelisiniz:

```ini
[Interface]
# ... diğer ayarlar ...
Table = off
```

Bu komut WireGuard'a şunu söyler: *"Tüneli aç, şifreleme rotalarını hazırla ama Linux ana yönlendirme tabloma (routing table) sakın dokunma! O işi BGP halledecek."*

## 4. FRRouting (FRR) ile BGP Konfigürasyonu

Modern Linux sistemlerinde BGP çalıştırmak için en popüler yazılım **FRRouting (FRR)**'dir.

**Senaryo:** Merkez (HQ) ve Şube 1 arasında BGP ile dinamik rota alışverişi.

### A. WireGuard Tünel IP'leri (Transit Network)

Her iki tarafın sadece birbiriyle konuşacağı küçük bir transit ağ (`/30`) tanımlanır.

- Merkez `wg1` IP: `10.255.0.1/30`
- Şube `wg1` IP: `10.255.0.2/30`

### B. FRR BGP Yapılandırması (Merkez)

FRR konsoluna (`vtysh`) girilip şu tanımlar yapılır:

```text
router bgp 65000
 bgp router-id 10.255.0.1
 neighbor 10.255.0.2 remote-as 65001
 neighbor 10.255.0.2 description Sube_1
 
 address-family ipv4 unicast
  network 192.168.10.0/24  <-- Kendi yerel ağımızı anons ediyoruz
  neighbor 10.255.0.2 route-map Sube1_Filtre in
 exit-address-family
```

### C. Dinamik Keşif

Şube 1, FRR üzerinden kendi yerel ağı olan `192.168.20.0/24`'ü anons ettiğinde, Merkez'deki FRR yazılımı bu anonsu WireGuard içinden gelen TCP 179 paketleriyle alır. FRR, bu rotayı otomatik olarak Linux Kernel Routing tablosuna ekler:
`192.168.20.0/24 dev wg1 proto bgp`

Şubeye yeni bir alt ağ eklendiğinde, WireGuard tarafında **hiçbir değişiklik yapmanıza gerek kalmaz**. BGP saniyeler içinde yeni rotayı Merkez'e öğretir.

## 5. Güvenlik ve Route Filtering (Rota Filtreleme)

WireGuard'da `AllowedIPs = 0.0.0.0/0` ayarladığımız için, tünelin içine kimlik avı veya hatalı rota basma (Route Leak) riskleri doğar.
Kötü yapılandırılmış bir Şube, yanlışlıkla `0.0.0.0/0` (Default Route) anons ederek Merkezin tüm internetini kesebilir.

Bunu önlemek için FRR içinde daima **Prefix-List** ve **Route-Map** kullanılmalıdır:

```text
ip prefix-list Sube1_Izinli_Aglari seq 10 permit 192.168.20.0/24
ip prefix-list Sube1_Izinli_Aglari seq 20 deny any

route-map Sube1_Filtre permit 10
 match ip address prefix-list Sube1_Izinli_Aglari
```

Bu sayede Şube 1'den WireGuard tüneli üzerinden sadece izin verilen IP bloklarına ait BGP anonsları kabul edilir.

---

**Sonuç:** BGP ve WireGuard, modern ağ mühendisliğinin rüya takımıdır. WireGuard kusursuz, hafif ve kriptografik bir veri düzlemi (Data Plane) sağlarken; FRR/BGP mükemmel, standartlara uygun bir kontrol düzlemi (Control Plane) sağlar.

[<< Önceki Bölüm](10_paket_yolculugu.md) | [Ana Sayfa](README.md) | [Sonraki Bölüm >>](12_performans_optimizasyonu.md)
