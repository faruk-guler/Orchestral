# Useful Wireshark Filters Cheatsheet

```text
✓ `ip.addr == 10.0.0.1` 10.0.0.1 adresini kaynak veya hedef olarak içeren tüm trafiği göster
✓ `ip.addr == 10.0.0.0/24` 10.0.0.0/24 ağındaki herhangi bir adrese gelen veya giden tüm trafiği göster
✓ `ip.src == 10.0.0.1 && ip.dst == 10.0.0.2` 10.0.0.1'den 10.0.0.2'ye giden tüm trafiği göster
✓ `!(ip.addr == 10.0.0.1)` 10.0.0.1'e veya 10.0.0.1'den gelen tüm trafiği hariç tut
✓ `ip.ttl < 10` TTL değeri 10'dan küçük paketleri göster (yönlendirme döngülerini tespit etmek için kullanışlı)
✓ `icmp.type == 3` ICMP "hedefe ulaşılamıyor" paketlerini göster
✓ `tcp or udp` TCP veya UDP trafiğini göster
✓ `tcp.port == 80` Port 80 üzerindeki TCP trafiğini göster
✓ `tcp.srcport < 1000` Kaynak portu 1000'den küçük TCP trafiğini göster
✓ `!(tcp or udp)` TCP/UDP dışı trafiği bul (alışılmadık protokoller)
✓ `http or dns` Tüm HTTP veya DNS trafiğini göster
✓ `tcp.flags.syn == 1` SYN bayrağı set edilmiş TCP paketlerini göster
✓ `tcp.flags == 0x012` Hem SYN hem de ACK bayrağı set edilmiş TCP paketlerini göster
✓ `tcp.analysis.retransmission` Yeniden iletilen tüm TCP paketlerini göster
✓ `tcp.analysis.lost_segment` Kayıp olarak işaretlenen TCP segmentlerini göster
✓ `tcp.analysis.duplicate_ack` Paket kaybına işaret eden tekrarlı TCP ACK'lerini göster
✓ `http.request.method == "GET"` HTTP GET isteğiyle ilişkili paketleri göster
✓ `http.response.code == 404` HTTP 404 yanıtıyla ilişkili paketleri göster
✓ `http.host == "www.test.com"` Host başlık alanıyla eşleşen HTTP trafiğini göster
✓ `tls.handshake` Yalnızca TLS el sıkışma paketlerini göster
✓ `tls.handshake.type == 1` TLS el sıkışması sırasındaki Client Hello paketini göster
✓ `dhcp and ip.addr == 10.0.0.0/24` 10.0.0.0/24 alt ağı için DHCP trafiğini göster
✓ `dhcp.hw.mac_addr == 00:11:22:33:44:55` Belirtilen istemci MAC adresi için DHCP paketlerini göster
✓ `dns.qry.name contains "cnn.com"` "cnn.com" içeren DNS sorgu paketlerini göster
✓ `dns.resp.name contains "cnn.com"` "cnn.com" içeren DNS yanıt paketlerini göster
✓ `dns.qry.name matches "\.xyz$|\.club$"` Belirtilen TLD'lerle biten DNS sorgu paketlerini göster
✓ `frame contains "keyword"` "keyword" kelimesini içeren tüm paketleri göster
✓ `frame.len > 1000` Toplam uzunluğu 1000 byte'tan büyük olan tüm paketleri göster
✓ `eth.addr == 00:11:22:33:44:55` Belirtilen MAC adresine gelen veya giden tüm trafiği göster
✓ `eth[0x47:2] == 01:80` 0x47 ofsetinde 2 byte == 01:80 olan Ethernet çerçevelerini eşleştir
✓ `!(arp or icmp or stp)` ARP, ICMP ve STP arka plan trafiğini filtrele
✓ `vlan.id == 100` VLAN ID 100 ile etiketlenmiş paketleri göster
```
