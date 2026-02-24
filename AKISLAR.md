# Platform Akışları – Yorum ve Reklam Sayfaları

Bu belge, Yemeksepeti, Trendyol, Getir yorum sayfaları ve Yemeksepeti reklam sayfası için adım adım akışları tanımlar.

---

## Yemeksepeti – Birleşik Akış (Yorum + Reklam)

Yemeksepeti yorum ve reklam **tek akış içinde** çalışır; ayarlar `YS_ADS_ENABLED` ile ayrılır.

- **Session**: `withYsSession` (tek browser context)
- **Storage state**: `config.ysStorageStatePath` (aynı dosya: `./data/storage-state.json`)
- **Kimlik bilgileri**: `YS_EMAIL`, `YS_PASSWORD`
- **Reklam modülü**: `YS_ADS_ENABLED=false` ile kapatılır

Ana döngü sırası:
1. Yorum + reklam taraması (birleşik YS session)
2. Session keepalive
3. Telegram komutları
4. Heartbeat (“Sistem ayakta”)
5. YS Ads bütçe kontrolü
6. YS Ads Telegram komutları

Yorum akışı ilk çalışır; oturum açılıp saklanır. Ardından reklam akışı aynı session ile promotion sayfasına gider. Giriş ekranı görünürse `ensureAuthenticated` tekrar çalışır, aksi halde mevcut oturum kullanılır.

---

## 1. Yemeksepeti – Yorum Sayfası Akışı

### Adımlar

1. **Sayfa açma**  
   `YS_REVIEWS_URL` (örn. `https://partner-app.yemeksepeti.com/.../reviews`) açılır.

2. **Login kontrolü**  
   - `withYsSession` → `safeGoto(targetUrl)` ile sayfaya gidilir.  
   - `ensureAuthenticated(page, context, { redirectUrl: authRedirectUrl })`:
     - Login ekranı görünürse: email/password doldurulur, Giriş tıklanır.  
     - URL’in `/login` veya `/auth` içermediği kontrol edilir.  
     - Oturum `config.ysStorageStatePath`’e kaydedilir.

3. **Yorum sayfasına geçiş**  
   - Zaten reviews URL’indeyse doğrudan devam.  
   - `ensureOutletFromUrl`: Marka/outlet seçimi (outlet kodu URL’de varsa).  
   - `ensureCommentsOnlyFilter`: “Yalnızca Yorumları Göster” filtresi.  
   - `setPageSizeTo50`, `goToFirstPage`: sayfa boyutu ve ilk sayfa.

4. **Yorumları toplama**  
   - `collectReviewsFromPages`: kartları parse eder, tarih/metin/cevap durumunu alır.  
   - `DATE_RE` ile geçerli tarih formatı kontrol edilir.

---

## 2. Yemeksepeti – Reklam Sayfası Akışı

### Adımlar

1. **Sayfa açma**  
   Promotion sayfası (örn. `https://partner-app.yemeksepeti.com/promotion/premium-placement`) açılır.

2. **Login kontrolü**  
   - `withYsSession` aynı oturumu kullanır.  
   - `gotoAds()` ile ads URL’e gidilir.  
   - `isLikelyLoginScreen(page)`:
     - Login ekranı varsa: `ensureAuthenticated(page, context, { redirectUrl: adsUrl })`  
     - Sonrasında tekrar `gotoAds()`.

3. **Reklam sayfasına geçiş**  
   - Sound Check / Robot modal’ları kapatılır.  
   - `isOnAdsPage(url)`: URL `/advertising/`, `/promotion/` veya `campaign` içeriyor mu kontrol edilir.  
   - `maybeRecoverFromDashboardRedirect`: Dashboard’da kalındıysa ads linkine tıklanır.  
   - Landing’deyse: `tryNavigateFromLandingToCampaign` → “Detayları görüntüle” vb. ile kampanya detay sayfasına gidilir.

4. **Bütçe parse**  
   - `parseSnapshotFromPage`: totalBudget, spentBudget, remainingBudget okunur.  
   - Landing’de sadece Maliyet varsa navigasyon retry + tekrar parse denenir.

---

## 3. Trendyol – Yorum Sayfası Akışı

### Adımlar

1. **Sayfa açma**  
   `TRENDYOL_REVIEW_URL` açılır (login yönlendirmesi olabilir).

2. **Login kontrolü**  
   - Ayrı session: `config.trendyolStorageStatePath`.  
   - `ensureTrendyolAuthenticated(page, context)` (Trendyol’a özel login).

3. **Yorum sayfasına geçiş**  
   - URL farklıysa review URL’e tekrar gidilir.  
   - `ensureTrendyolPendingFilters`: bekleyen yorum filtresi.

4. **Yorumları toplama**  
   - `bl-table-row[selection-key]` ile kartlar bulunur.  
   - Tarih, müşteri adı, yorum metni parse edilir.

---

## 4. Getir – Yorum Sayfası Akışı

### Adımlar

1. **Sayfa açma**  
   `GETIR_REVIEW_URL` açılır.

2. **Login kontrolü**  
   - Ayrı session: `config.getirStorageStatePath`.  
   - `ensureGetirAuthenticated(page, context)`.

3. **Yorum sayfasına geçiş**  
   - `openGetirReviewsPage`: SPA view state kontrol edilir.  
   - `isGetirReviewsView(state)` değilse “Yorumlar” linki tıklanır.  
   - Gerekirse `reviewUrl`’e tekrar gidilip login tekrarlanır.

4. **Yorumları toplama**  
   - Getir SPA view state üzerinden yorum kartları parse edilir.

---

## Özet – Session Paylaşımı

| Platform   | Yorum sayfası       | Reklam sayfası | Ortak oturum? |
|-----------|---------------------|----------------|---------------|
| Yemeksepeti | `withYsSession`     | `withYsSession`| Evet          |
| Trendyol  | Ayrı browser/context| –              | Hayır         |
| Getir     | Ayrı browser/context| –              | Hayır         |

Yemeksepeti için: önce yorum akışı çalışır, session açılır ve kaydedilir; ardından reklam akışı aynı session ile promotion sayfasına gider ve giriş ekranı varsa tekrar authenticate eder.

---

## Docker

Aynı akışlar Docker container içinde de geçerlidir. Ayarlar host’taki `.env` dosyasından okunur; oturum ve state `./data` volume’u ile kalıcı tutulur. Detay için [DOCKER.md](DOCKER.md) belgesine bakın.
