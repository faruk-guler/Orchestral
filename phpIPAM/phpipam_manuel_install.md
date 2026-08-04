# phpIPAM on Debian 13 (Trixie) Nginx + PHP-FPM + MariaDB

**phpIPAM**, web tabanlı, açık kaynaklı ve modüler bir IP Adres Yönetim (IPAM - IP Address Management) yazılımıdır.

Bu rehber, **Debian 13 (Trixie)** üzerinde en yüksek performansı sağlayan **Nginx + PHP-FPM 8.4 + MariaDB** kombinasyonunu kullanarak phpIPAM'i baştan sona eksiksiz şekilde kurmanız için hazırlanmıştır.

---

## 📘 Adım Adım Manuel Kurulum

### Adım 1: Sistem Güncellemeleri ve Paket Kurulumları

Sistem paketlerinizi güncelleyin, zaman dilimini ayarlayın ve Nginx, MariaDB, PHP-FPM ve bağımlılıkları yükleyin:

```bash
# 1.1 Sistem güncellemesi ve Zaman Dilimi
sudo apt update && sudo apt full-upgrade -y
sudo timedatectl set-timezone Europe/Istanbul

# 1.2 Nginx, MariaDB, fping, SNMP, UFW ve Temel Araçlar
sudo apt install -y nginx mariadb-server mariadb-client git curl wget unzip fping snmp snmpd ufw build-essential ca-certificates lsb-release

# 1.3 PHP-FPM ve phpIPAM için Gerekli PHP Eklentileri
sudo apt install -y php-fpm php-cli php-common php-mysql php-gmp php-gd php-curl \
  php-intl php-mbstring php-xml php-zip php-snmp php-bcmath php-json

# 1.4 Servisleri Başlatın ve Otomatik Açılışa Ekleyin
sudo systemctl enable --now nginx mariadb php8.4-fpm snmpd
```

---

### Adım 2: PHP-FPM Ayarları (php.ini)

phpIPAM'in büyük IP bloklarını ve dosya içe/dışa aktarımlarını sorunsuz işleyebilmesi için PHP limitlerini düzenleyin:

```bash
sudo nano /etc/php/8.4/fpm/php.ini
```

Aşağıdaki değerleri güncelleyin/doğrulayın:

```ini
memory_limit = 256M
upload_max_filesize = 32M
post_max_size = 32M
max_execution_time = 300
date.timezone = Europe/Istanbul
```

PHP-FPM servisini yeniden başlatın:

```bash
sudo systemctl restart php8.4-fpm
```

---

### Adım 2: Veritabanı (MariaDB) Yapılandırması

#### 2.1 MariaDB Güvenlik Sıkılaştırma

```bash
sudo mariadb-secure-installation
```

*(Sorulara: Root şifresi belirlemek için `Y`, kalan tüm sorulara `Y` yanıtını verin).*

#### 2.2 Veritabanı ve Kullanıcı Oluşturma

MariaDB konsoluna bağlanın:

```bash
sudo mariadb -u root -p
```

SQL komutlarını sırasıyla çalıştırın *(Örnek `GuvenliSifreniz123!` şifresini kendi güçlü şifrenizle değiştirin)*:

```sql
CREATE DATABASE phpipam CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'phpipamuser'@'localhost' IDENTIFIED BY 'GuvenliSifreniz123!';
GRANT ALL PRIVILEGES ON phpipam.* TO 'phpipamuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

### Adım 3: phpIPAM Kodlarının İndirilmesi ve İzinler

```bash
# 3.1 GitHub Deposunu Klonlayın
cd /var/www
sudo git clone --recursive https://github.com/phpipam/phpipam.git ipam

# 3.2 Kararlı Dala Geçin
cd /var/www/ipam
sudo git checkout master
sudo git submodule update --init --recursive

# 3.3 İzinleri ve Sahipliği Ayarlayın (Debian Nginx kullanıcısı: www-data)
sudo chown -R www-data:www-data /var/www/ipam
sudo find /var/www/ipam -type d -exec chmod 755 {} \;
sudo find /var/www/ipam -type f -exec chmod 644 {} \;
```

---

### Adım 4: phpIPAM Konfigürasyonu (`config.php`)

```bash
cd /var/www/ipam
sudo cp config.dist.php config.php
sudo nano config.php
```

Veritabanı bağlantı bloklarını güncelleyin:

```php
$db['host'] = 'localhost';
$db['user'] = 'phpipamuser';
$db['pass'] = 'GuvenliSifreniz123!';
$db['name'] = 'phpipam';
$db['port'] = 3306;

// Debian 13 PHP 8.4 uyumluluk bypass'ı (phpIPAM 1.7.4 henüz PHP 8.4'ü resmi desteklemiyor)
$allow_untested_php_versions=true;
```

*(Not: Dizideki anahtar `$db['pass']` olmalıdır).*

Kaydedip çıkın (`Ctrl + O`, `Enter`, `Ctrl + X`).

---

### Adım 5: Nginx Web Sunucusu Yapılandırması

Nginx konfigürasyon dosyasını oluşturun:

```bash
sudo nano /etc/nginx/sites-available/phpipam
```

Aşağıdaki **Debian 13 & PHP 8.4** uyumlu Nginx blok içeriğini yapıştırın:

```nginx
server {
    listen 80;
    server_name ipam.domaininiz.com; # Veya sunucu IP adresiniz

    root /var/www/ipam;
    index index.php index.html;

    client_max_body_size 32M;

    access_log /var/log/nginx/phpipam_access.log;
    error_log  /var/log/nginx/phpipam_error.log;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_read_timeout 300;
        include fastcgi_params;
    }

    # API uç noktaları için yönlendirme
    location /api/ {
        try_files $uri $uri/ /api/index.php?$args;
    }

    # Gizli dosyalara ve hassas klasörlere erişimi engelle
    location ~ /\.ht {
        deny all;
    }

    location ~ /(app|config)/ {
        deny all;
    }
}
```

**Siteyi Etkinleştirin ve Test Edin:**

```bash
sudo ln -sf /etc/nginx/sites-available/phpipam /etc/nginx/sites-enabled/ipam
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

### Adım 6: Web Arayüzü ile Kurulumu Tamamlama

1. Tarayıcınızdan `http://SUNUCU_IP/` adresine gidin.
2. Ekranda **New phpipam installation** butonuna tıklayın.
3. **Automatic database installation** seçeneğini seçin.
4. Veritabanı bilgilerini girin:
   - **MySQL/MariaDB Username:** `phpipamuser`
   - **MySQL/MariaDB Password:** `GuvenliSifreniz123!`
   - **Database Name:** `phpipam`
5. Kurulum tamamlandığında Admin şifrenizi belirleyin (Varsayılan kullanıcı adı: `admin`).

---

### Adım 7: Kurulum Sonrası Kritik Güvenlik Sıkılaştırması

Web arayüzü ile veritabanı kurulumu tamamlandıktan sonra, güvenlik için `install` dizinini silin ve `config.php` dosyasına kurulum kilidini ekleyin:

```bash
# 1. Install dizinini silin
sudo rm -rf /var/www/ipam/install

# 2. config.php dosyasına kurulum kilit bayrağını ekleyin
echo "\$disable_installer = true;" | sudo tee -a /var/www/ipam/config.php
```

---

### Adım 8: Otomatik Taramalar (fping SUID & Cron Jobs)

#### 8.1 fping SUID İzni

```bash
sudo chmod +s /usr/bin/fping
```

#### 8.2 Cron Görevlerini Tanımlama

`www-data` kullanıcısı için crontab düzenleyin:

```bash
sudo crontab -e -u www-data
```

Gelen dosyaya ekleyin:

```cron
# Her 15 dakikada bir IP durum taraması yap (Ping)
*/15 * * * * /usr/bin/php /var/www/ipam/functions/scripts/pingCheck.php > /dev/null 2>&1

# Her 15 dakikada bir yeni cihaz keşfet (Discovery)
*/15 * * * * /usr/bin/php /var/www/ipam/functions/scripts/discoveryCheck.php > /dev/null 2>&1

# Her gece yarısı DNS çözümlemesi yap
0 0 * * * /usr/bin/php /var/www/ipam/functions/scripts/resolveIPaddresses.php > /dev/null 2>&1
```

#### 8.3 Panel İçi Ping ve FPing Yol Ayarları (Kritik)

Arayüzden otomatik taramaların çalışabilmesi için panel içerisinden binary yollarını doğrulayın:

1. Web panelinde **Administration > phpIPAM settings** menüsüne gidin.
2. Ayarları şu şekilde güncelleyin:
   - **Prettify links:** `Yes`
   - **Ping path:** `/usr/bin/ping`
   - **FPing path:** `/usr/bin/fping`
3. Ayarları kaydedin.

#### 8.4 Otomatik Veritabanı Yedekleme Cron'u (Önerilen)

Cron satırlarında ve işlem listesinde (`ps aux`) şifrenin düz metin olarak görünmemesi için şifrelerinizi bir konfigürasyon dosyasına yazın:

```bash
sudo nano /root/.my.cnf
```

Aşağıdaki bilgileri girin ve kaydedin:

```ini
[client]
user=phpipamuser
password=GuvenliSifreniz123!
```

Dosya izinlerini sadece root okuyabilecek şekilde daraltın:

```bash
sudo chmod 600 /root/.my.cnf
```

Ardından root kullanıcısının crontab'ına (`sudo crontab -e`) cron görevlerini ekleyin:

```cron
# Her gece saat 02:00'de yedek al ve 10 günden eski yedekleri otomatik temizle
0 2 * * * /usr/bin/mariadb-dump phpipam > /var/backups/phpipam_$(date +\%F).sql
0 3 * * * /usr/bin/find /var/backups/ -name "phpipam_*.sql" -mtime +10 -exec rm {} \;
```

---

### Adım 9: Güvenlik Duvarı (UFW) Yapılandırması

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

---

### Adım 10: HTTPS / SSL Sertifikası (Certbot)

Domain adınız aktif ise ücretsiz Let's Encrypt SSL sertifikasını etkinleştirin:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d ipam.domaininiz.com
```

---

## 💡 Dikkat Edilmesi Gereken İpuçları & Düzeltmeler

1. **`config.php` Konumu:** phpIPAM'in güncel sürümlerinde dosya kök dizindedir (`/var/www/ipam/config.php`).
2. **`$db['pass']` Değişkeni:** Config dosyasında şifre değişkeni `$db['password']` değil, `$db['pass']` şeklindedir.
3. **Nginx FastCGI Pass:** Debian 13 varsayılan PHP 8.4 kullandığı için Nginx içerisinde `unix:/run/php/php8.4-fpm.sock` yazılmalıdır.
4. **Hızlı (Çoklu-Thread) Tarama İpucu:** Arka plan taramalarının daha hızlı (multi-thread) çalışabilmesi için PHP'nin `posix` ve `pcntl` eklentilerine ihtiyacı vardır. `php -m | grep -E 'posix|pcntl'` komutu ile kontrol edebilir, eksikse `php-cli` yapılandırmasını inceleyebilirsiniz.

*Tebrikler! Debian 13 üzerinde Nginx + phpIPAM kurulumunuz başarıyla tamamlanmıştır.*

---
Author: faruk-guler
GitHub: [github.com/faruk-guler](https://github.com/faruk-guler)
Page: [www.farukguler.com](https://www.farukguler.com)
