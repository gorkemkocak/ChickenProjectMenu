# Docker ile Çalıştırma

Bu belge, Restoran Paneli uygulamasını Docker ve Docker Compose ile çalıştırma adımlarını ve ayar detaylarını açıklar.

---

## Genel Bakış

- **Uygulama imajı:** Playwright (Chromium) tabanlı `mcr.microsoft.com/playwright:v1.56.0-jammy` kullanılır; ek tarayıcı kurulumu gerekmez.
- **Servisler:** `app` (Node.js + Playwright) ve `mongo` (MongoDB 6).
- **Ayarlar:** Container içinde hem `env_file: .env` hem de host’taki `.env` dosyası `/app/.env` olarak bağlanır; tüm panel ayarları (Telegram, YS, Trendyol, Getir, Mongo vb.) buradan okunur.
- **Veri:** Oturum ve state dosyaları `./data`, MongoDB verisi `./mongo-data` ile host’a bağlanır.

---

## Gereksinimler

- Docker ve Docker Compose kurulu olmalı.
- Proje kökünde `.env` dosyası bulunmalı (örnek için `cp .env.example .env`).

---

## Önemli: Proje Kökünden Çalıştırma

Compose **her zaman proje kökünden** çalıştırılmalıdır. Böylece:

- `env_file: .env` ve volume `./.env:/app/.env:ro` doğru dizindeki `.env` dosyasını kullanır.
- `./data` ve `./mongo-data` volume’ları aynı dizindeki klasörlere bağlanır.

Örnek:

```bash
cd /path/to/RestoranPaneli
docker compose up -d
```

Başka bir dizinden `docker compose` çalıştırırsanız ayarlar ve veri yolları yanlış olur; container içinde ayarlar “boş” görünebilir.

---

## Compose Yapılandırması Özeti

| Öğe | Açıklama |
|-----|----------|
| **env_file** | `.env` – Compose, bu dosyadaki değişkenleri container ortamına enjekte eder. |
| **Volume: .env** | `./.env:/app/.env:ro` – Host’taki `.env` container’da okunur; uygulama `dotenv` ile de bu dosyayı kullanır. |
| **Volume: data** | `./data:/app/data` – Oturum (storage state), debug ve diğer runtime dosyaları kalıcı tutulur. |
| **Volume: mongo** | `./mongo-data:/data/db` – MongoDB veritabanı host’ta saklanır. |
| **Portlar** | `8787` (panel), `27017` (MongoDB). |

---

## Komutlar

### İlk kurulum ve çalıştırma

```bash
cd /path/to/RestoranPaneli
cp .env.example .env   # gerekirse: .env düzenle
docker compose up -d
```

Panel: **http://localhost:8787**

### Yeniden başlatma (ayar değişikliği sonrası)

```bash
docker compose down
docker compose up -d
```

### Imajı yeniden build etmek (kod değişikliği sonrası)

```bash
docker compose build app
docker compose up -d
```

### Logları izlemek

```bash
docker compose logs -f app
```

---

## Ortam Değişkenleri ve MongoDB

Container içinde:

- `MONGO_URL` genelde `mongodb://mongo:27017` olarak ayarlanır (Compose’taki `mongo` servisi).
- `MONGO_DB_NAME` ile veritabanı adı seçilir (örn. `restoran-paneli`).

Bu değişkenler `.env` içinde tanımlanmalı; böylece uygulama container’da MongoDB’ye bağlanır ve loglar Mongo’ya yazılır.

---

## Oturum ve Robot / CAPTCHA

- Tarayıcı container içinde headless çalışır; robot veya CAPTCHA sayfalarında manuel müdahale yapılamaz.
- **Yemeksepeti / Reklam oturumu tazeleme:** Gerekirse oturumu **host’ta** (Cursor veya terminalde) açarak yenileyin:
  ```bash
  cd /path/to/RestoranPaneli
  YS_HEADLESS=false npm run auth
  # veya reklam için: YS_ADS_HEADLESS=false npm run auth:ads
  ```
  Oturum `./data` içine yazılır; container aynı `./data` volume’unu kullandığı için bir sonraki çalışmada güncel oturum kullanılır.

---

## Sorun Giderme

### Container’da ayarlar boş görünüyor
- Compose’u **proje kökünden** çalıştırdığınızdan emin olun (`cd /path/to/RestoranPaneli`).
- Host’ta `.env` dosyasının var olduğunu ve volume’ların doğru bağlandığını kontrol edin:  
  `docker compose exec app cat /app/.env` (ilk satırlar görünmeli).

### MongoDB bağlantı hatası
- `mongo` servisinin ayakta olduğunu kontrol edin: `docker compose ps`.
- `.env` içinde `MONGO_URL=mongodb://mongo:27017` (ve varsa `MONGO_DB_NAME`) tanımlı olsun.

### Playwright / Chromium hatası
- Kullanılan imaj: `mcr.microsoft.com/playwright:v1.56.0-jammy`. Farklı bir sürüm kullanıyorsanız Dockerfile’daki `FROM` satırını bu sürümle eşleştirin.

---

## İlgili Belgeler

- [README.md](../README.md) – Genel kurulum ve kullanım
- [REKLAM-KILAVUZU.md](REKLAM-KILAVUZU.md) – Reklam bütçe otomasyonu ve Docker notu
- [AKISLAR.md](AKISLAR.md) – Platform akışları (Docker’da aynı akışlar geçerlidir)
