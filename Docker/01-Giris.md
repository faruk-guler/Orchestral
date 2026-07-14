# Bölüm 1: Docker'a Giriş ve Konteyner Mimarisi

Yazılım geliştirme dünyasında en sık karşılaşılan sorunlardan biri şudur: *"Benim bilgisayarımda çalışıyordu, sunucuda neden çalışmıyor?"* 

İşte Docker, tam olarak bu sorunu çözmek için ortaya çıkmış bir teknolojidir. Bu bölümde Docker'ın ne olduğunu, neden bu kadar popülerleştiğini ve standart bir kullanıcı seviyesinden çıkarak arka planda dönen derin teknik mimariyi "Zero to Hero" vizyonuyla ele alacağız.

## Docker Nedir?

Docker, uygulamalarınızı ve bu uygulamaların çalışması için gereken tüm bağımlılıkları (kütüphaneler, ayarlar, sistem araçları) bir **konteyner (container)** içine paketlemenizi sağlayan açık kaynaklı bir platformdur. 

Bir uygulamayı konteynerleştirdiğinizde, uygulamanızın nerede çalıştığı (sizin laptop'ınız, şirketinizin test sunucusu veya AWS/Google Cloud üzerindeki bir sunucu) fark etmeksizin **her zaman aynı şekilde** çalışacağından emin olursunuz.

### Neden Docker Kullanmalıyız?

1. **Tutarlılık:** Geliştirme, test ve canlı (production) ortamları arasındaki farklılıkları tamamen ortadan kaldırır. "Works on my machine" (Benim makinemde çalışıyor) bahanesini tarihe gömer.
2. **Hız:** Konteynerler saniyeler, hatta milisaniyeler içinde başlatılabilir, çünkü geleneksel sistemlerin aksine bir işletim sistemi önyükleme (boot) sürecine ihtiyaç duymazlar.
3. **Taşınabilirlik:** Bir kere build ettiğiniz (inşa ettiğiniz) bir Docker imajı, Docker yüklü olan Linux, Windows veya macOS fark etmeksizin her yerde çalışabilir.
4. **İzolasyon:** Her konteyner kendi yalıtılmış ortamında çalışır. Bir konteynerdeki sorun veya bağımlılık diğerini etkilemez. (Örneğin aynı makinede biri Node.js 14, diğeri Node.js 18 kullanan iki projeyi hiçbir çakışma olmadan yan yana çalıştırabilirsiniz).
5. **Kaynak Verimliliği:** Sanal makinelere göre devasa oranda daha az kaynak (CPU/RAM/Disk) tüketir.

---

## Sanal Makineler (VM) vs Konteynerler

Docker'ı daha iyi anlamak için, ondan önceki endüstri standardı olan Sanal Makineler (Virtual Machines - VM) ile karşılaştırmak en doğru yöntemdir.

### Sanal Makine (VM) Yaklaşımı
Geleneksel sanallaştırmada, bir sunucu üzerine **Hypervisor** (VMware, VirtualBox, Hyper-V vb.) kurulur. Bu hypervisor üzerinde oluşturulan her Sanal Makine kendi **Misafir İşletim Sistemine (Guest OS)** ihtiyaç duyar.
- **Ağırlık:** Çok ağırdır (GB'larca disk alanı kaplar, her VM kendi Kernel'ına sahiptir).
- **Hız:** Başlatılması dakikalar sürer (Gerçek bir bilgisayar gibi boot olur).
- **Maliyet:** Her bir VM için ayrı işletim sistemi lisansı ve bakımı (güncellemeler, yama yönetimi) gerekir.
- **Kaynak Kullanımı:** Kaynaklar (RAM/CPU) statik olarak atanır. Bir VM'e 8GB RAM atadıysanız, uygulama sadece 1GB kullansa bile kalan 7GB RAM host sistemden bloke edilir.

### Konteyner (Docker) Yaklaşımı
Konteynerler ise fiziksel (host) makinenin **çekirdeğini (kernel)** doğrudan paylaşırlar. Docker Engine üzerinde koşan konteynerlerin kendi misafir işletim sistemleri yoktur.
- **Ağırlık:** Çok hafiftir (MB'lar, hatta Alpine Linux gibi imajlarda KB'lar mertebesinde olabilir).
- **Hız:** Saniyeler içinde başlar, çünkü boot edilecek bir OS Kernel yoktur.
- **Verimlilik:** Ana makinenin Kernel'ini paylaştıkları için, kaynaklar sadece anlık ihtiyaç kadar kullanılır. Boşta yatan kaynaklar sisteme geri döner.

> [!TIP]
> Kısacası VM'ler **donanımı** (hardware) sanallaştırırken, Docker ve Konteynerler **işletim sistemini** (OS) sanallaştırır.

---

## İleri Seviye: Docker Arka Planda Nasıl Çalışır? (Under the Hood)

Docker aslında sihirli bir teknoloji değildir; Linux işletim sisteminin yıllardır sahip olduğu çekirdek özelliklerini (kernel features) bir araya getirip kullanımı inanılmaz kolay bir API sunan bir araçtır. Docker'ın belkemiğini oluşturan 4 temel Linux Kernel teknolojisi şunlardır:

### 1. Namespaces (İzolasyon)
Namespaces, bir konteynerin sistemin geri kalanından izole edilmesini sağlar. Bir konteyner çalıştığında Docker şu namespace'leri oluşturur:
- **PID (Process ID):** Konteynerin kendi süreç ağacı (process tree) olur. Konteyner içindeki 1 numaralı süreç (PID 1), dışarıdaki host sistemde aslında PID 4598 gibi bambaşka bir süreçtir. Konteyner dışarıyı göremez.
- **NET (Networking):** Konteynerin kendi sanal ağ arabirimi (eth0), IP adresi ve port yönlendirme tablosu olur.
- **MNT (Mount):** Konteyner kendi izole edilmiş dosya sistemine sahiptir. Host sistemin kök dizinini (`/`) göremez.
- **IPC (InterProcess Communication):** Konteyner içindeki süreçlerin paylaşımlı belleklere erişimini izole eder.
- **UTS (UNIX Timesharing System):** Konteynerin kendine ait izole bir *hostname* (bilgisayar adı) olmasını sağlar.
- **USER:** Host makinedeki root kullanıcısı ile konteynerdeki root kullanıcısını ayırmayı sağlar (Güvenlik için kritik bir özelliktir).

### 2. Cgroups (Control Groups - Kaynak Sınırlandırma)
Namespaces izolasyonu sağlarken, **cgroups** kaynak tüketimini (Resource Limiting) kontrol eder. 
Eğer cgroups olmasaydı, bir konteyner hatalı bir kod yüzünden host makinedeki tüm RAM'i (memory leak) veya CPU'yu tüketip bütün sunucuyu çökertebilirdi. Cgroups sayesinde Docker'a "Bu konteyner maksimum 512MB RAM ve CPU'nun sadece %50'sini kullanabilsin" diyebiliriz.

### 3. Union File System (UnionFS / Overlay2)
Docker imajları tek bir devasa dosya değildir; katmanlardan (layers) oluşur. UnionFS, birden fazla dosya sisteminin üst üste bindirilip (overlay) tek bir dosya sistemiymiş gibi görünmesini sağlayan teknolojidir. 
Bir Ubuntu imajının üzerine Nginx kurduğunuzda, Nginx sadece yeni bir katman olarak üste eklenir. İmajı indiren başka biri, Ubuntu katmanına zaten sahipse sadece Nginx katmanını indirir. Bu, muazzam bir disk ve ağ verimliliği sağlar.

### 4. chroot (Kök Dizini Değiştirme)
Linux'taki `chroot` komutunun gelişmiş bir versiyonu kullanılarak, konteynerin kök dizini (root directory) değiştirilir. Konteyner içindeki bir süreç `cd /` yazdığında, aslında host makinenin kök dizinine değil, Docker'ın ona tahsis ettiği izole klasöre gider.

---

## Docker'ın Temel Mimarisi (İstemci-Sunucu)

Docker, İstemci-Sunucu (Client-Server) mimarisine dayanır.

1. **Docker Daemon (dockerd):** Arka planda çalışan ağır işçidir. Konteynerleri, imajları, ağları ve veri birimlerini (volumes) yönetir. Doğrudan Kernel (namespaces, cgroups) ile o konuşur.
2. **Docker Client (docker CLI):** Sizin terminalden `docker run` veya `docker pull` komutlarını yazdığınız komut satırı aracıdır. İstemci sadece bir arayüzdür; komutları alır ve Docker Daemon'a REST API (genellikle yerel UNIX soketi `/var/run/docker.sock` üzerinden) ileterek işlemi yaptırır.
3. **Docker Registry (Kayıt Defteri):** Docker imajlarının depolandığı, paylaşıldığı ve indirildiği (pull) merkezlerdir. En bilineni, varsayılan olarak kullanılan genel **Docker Hub**'dır. (GitHub'ın Docker imajları için olan versiyonu gibi düşünebilirsiniz).

### Önemli Terimler Sözlüğü

- **Image (İmaj):** Çalıştırılabilir, salt-okunur (read-only) paketlerdir. Uygulamanın kodunu, runtime'ını, kütüphanelerini ve bağımlılıklarını içerir. (Nesne Yönelimli Programlamadaki *Sınıf* / *Class*).
- **Container (Konteyner):** İmajın çalışan (veya durdurulmuş) canlı halidir. Üzerine yazılabilir (writable) ince bir katman eklenmiş halidir. (Nesne Yönelimli Programlamadaki *Nesne* / *Instance*).
- **Dockerfile:** Bir Docker imajının nasıl inşa edileceğini sıfırdan anlatan komut listesi / metin belgesidir. (İmajın kaynak kodudur).

---
*Giriş bölümüyle Docker'ın yüzeyini ve arka plandaki derin Kernel yeteneklerini (Namespaces, Cgroups) öğrendik. Sıradaki bölümde, Docker'ı farklı işletim sistemlerine nasıl kuracağımızı ve Daemon konfigürasyonlarını (Advanced Setup) inceleyeceğiz.*
