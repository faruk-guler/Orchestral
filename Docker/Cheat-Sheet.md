---
title: Docker Hızlı Referans (Cheat Sheet)
date: 2026-07-14
background: bg-[#488fdf]
tags:
  - container
  - virtual
categories:
  - Programming
intro: |
  Bu belge [Docker](https://docs.docker.com/get-started/) için hızlı bir başvuru kaynağıdır (Cheat Sheet). En sık kullanılan Docker komutlarını burada bulabilirsiniz.
plugins:
  - copyCode
---

## Başlangıç

### İlk Adımlar

Arka planda (background) bir konteyner oluşturup çalıştırmak:

```shell
docker run -d -p 80:80 docker/getting-started
```

---

- `-d` - Konteyneri arka planda (detached mode) çalıştırır
- `-p 80:80` - Host makinedeki 80 portunu konteynerin 80 portuna bağlar
- `docker/getting-started` - Kullanılacak imaj

Ön planda (foreground) etkileşimli bir konteyner çalıştırmak:

```shell
docker run -it -p 8001:80 --name my-nginx nginx
```

---

- `-it` - Etkileşimli terminal (bash) modunda çalıştırır
- `-p 8001:80` - Host portunu 8001'den konteynerin 80 portuna yönlendirir
- `--name my-nginx` - Konteynere özel bir isim atar
- `nginx` - Kullanılacak imaj

### Genel Komutlar

| Örnek                               | Açıklama                                         |
| ----------------------------------- | ------------------------------------------------ |
| `docker ps`                         | Çalışan konteynerleri listeler                   |
| `docker ps -a`                      | Bütün konteynerleri (duranlar dahil) listeler    |
| `docker ps -s`                      | Çalışan konteynerleri disk boyutlarıyla listeler |
| `docker images`                     | Tüm imajları listeler                            |
| `docker exec -it <konteyner> bash`  | Çalışan konteyner içinde terminal (bash/sh) başlatır |
| `docker logs <konteyner>`           | Konteynerin konsol kayıtlarını (loglarını) gösterir |
| `docker logs -f <konteyner>`        | Konteyner loglarını canlı takip eder (follow)    |
| `docker stop <konteyner>`           | Konteyneri durdurur                              |
| `docker restart <konteyner>`        | Konteyneri yeniden başlatır                      |
| `docker rm <konteyner>`             | Konteyneri siler                                 |
| `docker port <konteyner>`           | Konteynerin port eşleşmelerini gösterir          |
| `docker top <konteyner>`            | İçeride çalışan süreçleri (process) listeler     |
| `docker kill <konteyner>`           | Konteyneri anında sonlandırır (Kill)             |

`<konteyner>` parametresi, konteynerin ID'si veya ismi olabilir.

## Docker Konteynerleri

### Başlatma & Durdurma

| Örnek                       | Açıklama                            |
| ------------------------- | ----------------------------------- |
| `docker start my-nginx`   | Başlatma                            |
| `docker stop my-nginx`    | Durdurma                            |
| `docker restart my-nginx` | Yeniden Başlatma                    |
| `docker pause my-nginx`   | Duraklatma (Pause)                  |
| `docker unpause my-nginx` | Devam Ettirme (Unpause)             |
| `docker wait my-nginx`    | Konteyner kapanana kadar terminali bekletir |
| `docker kill my-nginx`    | SIGKILL sinyali gönderir            |
| `docker attach my-nginx`  | Çalışan konteynerin ana girdi/çıktı akışına (PID 1) bağlanır |

### Bilgi Alma

| Örnek                     | Açıklama                               |
| ------------------------- | -------------------------------------- |
| `docker ps`               | Çalışan konteynerleri listeler         |
| `docker ps -a`            | Tüm konteynerleri listeler             |
| `docker logs my-nginx`    | Konteyner Loglarını verir              |
| `docker inspect my-nginx` | Konteynerin tüm detaylarını (JSON) verir |
| `docker events --filter container=my-nginx` | Konteyner ile ilgili sistem olaylarını gerçek zamanlı izler |
| `docker port my-nginx`    | Dışarı açılan portları listeler        |
| `docker top my-nginx`     | Çalışan süreçleri listeler             |
| `docker stats my-nginx`   | Anlık kaynak tüketimini (RAM/CPU) gösterir |
| `docker diff my-nginx`    | Konteyner dosya sisteminde yapılan değişiklikleri gösterir |

### Oluşturma (Create)

```shell
docker create [secenekler] IMAJ
  -a, --attach               # stdout/err bağlar
  -i, --interactive          # stdin bağlar (etkileşimli)
  -t, --tty                  # pseudo-tty atar
      --name ISIM            # konteynere isim verir
  -p, --publish 5000:5000    # port yönlendirmesi (host:konteyner)
      --expose 5432          # portu sadece diğer konteynerlere açar
  -P, --publish-all          # EXPOSE edilen tüm portları rastgele host portlarına açar
      --network benim-agim   # konteyneri özel bir ağa bağlar (link yerine modern kullanım)
      --rm                   # konteyner durduğunda kendini otomatik siler
  -v, --volume $(pwd):/app   # klasör bağlar (mutlak yol olmalıdır)
  -e, --env NAME=hello       # ortam değişkeni (env) atar
```

#### Örnek

```shell
docker create --name my_redis --expose 6379 redis:3.0.2
```

### Yönetim (Manipulating)

Konteynerin İsmini Değiştirme

```shell
docker rename my-nginx yeni-nginx
```

Konteyneri Silme

```shell
docker rm my-nginx
```

Konteyneri Güncelleme (Kaynak Limitlerini)

```shell
docker update --cpus 1.5 -m 300M my-nginx
```

Dosya Kopyalama (Host -> Konteyner)

```shell
docker cp ./dosya.txt my-nginx:/app/dosya.txt
```

Dosya Kopyalama (Konteyner -> Host)

```shell
docker cp my-nginx:/app/dosya.txt ./dosya.txt
```

## Docker İmajları (Images)

### Yönetim

| Örnek                              | Açıklama                        |
| ---------------------------------- | ------------------------------- |
| `docker images`                    | İmajları listeler               |
| `docker rmi nginx`                 | İmajı siler                     |
| `docker load < ubuntu.tar.gz`      | Tar arşivinden imaj yükler      |
| `docker load --input ubuntu.tar`   | Tar arşivinden imaj yükler      |
| `docker save busybox > busybox.tar` | İmajı bir tar arşivi olarak kaydeder |
| `docker history nginx`             | İmajın katman (layer) geçmişini gösterir |
| `docker commit nginx my-image`     | Konteynerin mevcut durumunu yeni imaj yapar |
| `docker tag nginx sirketim/nginx`  | İmaja yeni bir etiket (isim) atar |
| `docker push sirketim/nginx`       | İmajı Docker Hub / Registry'e gönderir |

### İmaj İnşa Etmek (Build)

```shell
docker build .
docker build github.com/creack/docker-firefox
docker build - < Dockerfile
docker build - < context.tar.gz
docker build -t sirketim/my-nginx .
docker build -f baskaDockerfile .
docker build --build-arg VERSIYON=1.0 -t sirketim/my-nginx .
curl example.com/remote/Dockerfile | docker build -f - .
```

## Docker Ağları (Networking)

### Yönetim

Ağı Silme

```shell
docker network rm BenimOverlayAgim
```

Ağları Listeleme

```shell
docker network ls
```

Ağ Hakkında Detaylı Bilgi Alma

```shell
docker network inspect BenimOverlayAgim
```

Çalışan Konteyneri Ağa Bağlama

```shell
docker network connect BenimOverlayAgim nginx
```

Konteyneri Doğrudan Ağda Başlatma

```shell
docker run -it -d --network=BenimOverlayAgim nginx
```

Konteyneri Ağdan Koparma

```shell
docker network disconnect BenimOverlayAgim nginx
```

### Ağ Oluşturma

```shell
docker network create -d overlay BenimOverlayAgim

docker network create -d bridge BenimBridgeAgim

docker network create -d overlay \
  --subnet=192.168.0.0/16 \
  --subnet=192.170.0.0/16 \
  --gateway=192.168.0.100 \
  --gateway=192.170.0.100 \
  --ip-range=192.168.1.0/24 \
  BenimGelismisAgim
```

## Temizlik (Clean Up)

### Her Şeyi Temizleme

Dangling (boşta kalan) imajları, kullanılmayan konteynerleri ve ağları temizler (volume'leri varsayılan olarak temizlemez).

```shell
docker system prune
```

Kullanılmayan volume'ler dahil her şeyi temizlemek için:

```shell
docker system prune --volumes
```

---

Buna ek olarak durdurulmuş tüm konteynerleri ve kullanılmayan (bir konteynere bağlı olmayan) tüm imajları silmek için:

```shell
docker system prune -a
```

### Konteynerler

Çalışan tüm konteynerleri durdurma

```shell
docker stop $(docker ps -q)
```

Durdurulmuş konteynerleri silme

```shell
docker container prune
```

### İmajlar

Boşta kalan (Dangling - İsimsiz ve kullanılmayan) imajları silme:

```shell
docker image prune
```

Mevcut hiçbir konteyner tarafından kullanılmayan tüm imajları silme:

```shell
docker image prune -a
```

### Volume'ler

```shell
docker volume prune
```

En az bir konteyner tarafından kullanılmayan tüm volume'leri siler.

## Diğer (Miscellaneous)

### Docker Hub

| Docker Komutu               | Açıklama                            |
| --------------------------- | ----------------------------------- |
| `docker search arama_kelimesi` | Docker Hub üzerinde imaj arar.    |
| `docker pull user/image`    | Docker Hub'dan imaj indirir (pull). |
| `docker login`              | Docker Hub'a (veya registry'e) giriş yapar. |
| `docker push user/image`    | İmajı Docker Hub'a yükler.          |

### Registry Komutları

Registry'e Giriş

```shell
docker login
docker login localhost:8080
```

Registry'den Çıkış

```shell
docker logout
docker logout localhost:8080
```

İmaj Arama

```shell
docker search nginx
docker search --filter "stars=3" --no-trunc busybox
```

İmaj İndirme (Pull)

```shell
docker pull nginx
docker pull sirketim/nginx
docker pull localhost:5000/myadmin/nginx
```

İmaj Yükleme (Push)

```shell
docker push sirketim/nginx
docker push localhost:5000/myadmin/nginx
```

### Toplu İşlemler (Batch Clean)

| Örnek                               | Açıklama                |
| ----------------------------------- | ----------------------- |
| `docker stop $(docker ps -q)`       | Çalışan tüm konteynerleri durdurur |
| `docker rm -f $(docker ps -a -q)`   | Tüm konteynerleri zorla siler |
| `docker rmi -f $(docker images -q)` | Tüm imajları zorla siler |

### Volume'ler

Volume oluşturma:

```shell
docker volume create benim-volumem
```

Volume'leri listeleme:

```shell
docker volume ls
```

Volume hakkında detaylı bilgi alma:

```shell
docker volume inspect benim-volumem
```

Belirli bir volume'ü silme:

```shell
docker volume rm benim-volumem
```

Kullanılmayan tüm volume'leri temizleme:
 
 ```shell
 docker volume prune
 ```
 
## Docker Compose
 
Çoklu konteyner uygulamalarını tanımlamak ve çalıştırmak için kullanılan araçtır.
 
### Temel Komutlar
 
| Örnek | Açıklama |
| --- | --- |
| `docker compose up` | Servisleri ön planda (foreground) başlatır |
| `docker compose up -d` | Servisleri arka planda (detached) başlatır ve çalıştırır |
| `docker compose up -d --build` | İmajları yeniden derleyip servisleri başlatır |
| `docker compose down` | Servisleri durdurur; konteynerleri ve ağları siler |
| `docker compose down -v` | Servisleri durdurur; konteyner, ağ ve volume'leri siler |
| `docker compose ps` | Çalışan servisleri ve durumlarını listeler |
| `docker compose logs` | Servislerin log çıktılarını gösterir |
| `docker compose logs -f <servis>` | Belirtilen servisin loglarını canlı takip eder (follow) |
| `docker compose exec <servis> sh` | Çalışan servisin terminaline bağlanır |
| `docker compose restart` | Servisleri yeniden başlatır |
| `docker compose config` | Yapılandırma dosyasını doğrular ve inceler |
 
### Örnek `compose.yaml` Şablonu
 
```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: gizlisifre
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```
