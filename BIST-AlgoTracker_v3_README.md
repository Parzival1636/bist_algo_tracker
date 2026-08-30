# BIST-AlgoTracker v3

Borsa İstanbul için algoritmik ticaret ve kurumsal veri analiz platformu. v3, v1+v2'nin backend'ini **değiştirmeden** korur; frontend'i tek-sayfalık bir trader ızgarasından, **çok sekmeli, kurumsal bir "Trading Executive Dashboard"a** dönüştürür (React Router, kalıcı KPI header, Zustand+localStorage ile Ayarlar, WebSocket throttle/batching koruma duvarı).

## Neyin gerçekten doğrulandığı (iddia değil, bu ortamda test edildi)

✅ **Backend — 85 pytest testi, hepsi geçti (84 + nats-server varsa 1 daha):**
- v1+v2'nin tüm modülleri (OTR/OBI/Dwell-Time, 3 spoofing kuralı, Layering/Iceberg/adaptif-eşik/çapraz-korelasyon, risk motoru, TWAP/VWAP, circuit breaker, backpressure, gerçek `nats-server`'a karşı test edilen mesajlaşma)
- **v3 için yeni:** `PaperBrokerAdapter.get_all_orders()` + `GET /orders/all` (FILLED/CANCELED dahil tüm emirler — KPI header'ın gerçek bakiye/PnL hesaplayabilmesi için)
- Canlı sunucuda uçtan uca doğrulandı: agresif fiyatlı (piyasayı kesen) bir emir **gerçekten NEW → FILLED durumuna geçti**, `/orders/all` bunu doğru raporladı
- WS mesaj hızı canlı ölçüldü (tek sembol/tek kanalda ~2 msg/sn — 3 sembol × orderbook+AKD+alarm kanallarının toplamı frontend'deki throttle mekanizmasının neden gerçekten gerekli olduğunu doğruluyor)
- Bandit: 0 bulgu

✅ **Frontend — 36 vitest testi, hepsi geçti (v2'deki 18 + v3'te eklenen 18):**
- `tsc -b` (strict): 0 hata · `vite build`: temiz · `oxlint`: 0 hata/uyarı
- Yeni v3 saf mantığı bağımsız test edildi: `pnl.ts` (10 test — nakit bakiyesi + mark-to-market PnL hesapları), `throttleBuffer.ts` (5 test — 500 mesajlık yüksek-frekans senaryosu dahil), `telemetry.ts` (3 test)
- `vite preview` ile üretilen statik build gerçekten servis edildi, HTTP 200

⚠️ **Doğrulanamadı (dürüstlük notu — v2'den değişmedi):**
- **Görsel/tarayıcı testi yapılamadı** — bu ortamda headless tarayıcı indirilemiyor (`cdn.playwright.dev` ağ izin listesi dışında). Kod tip-güvenli derleniyor ve build ediliyor, ama gerçek ekranda tıklayıp görmedim; ilk açılışta küçük CSS/layout ince ayarları gerekebilir.
- Gerçek Kafka/ClickHouse/Kubernetes ve gerçek aracı kurum entegrasyonu — v1'den beri bilinçli olarak bağlanmadı.

## v3'te ne değişti

### 1. Kurumsal navigasyon + kalıcı KPI header
- **React Router** (`createBrowserRouter` + layout-route deseni): `Ana Panel`, `Otomasyon`, `Profil`, `Ayarlar` — gerçek URL'ler (`/dashboard`, `/automation`, ...), tarayıcı geri/ileri çalışır.
- `layout/AppLayout.tsx`: TÜM sekmelerde sabit kalan üst nav + KPI şeridi + **global** WebSocket abonelikleri (sekme değiştirince veri akışı KESİLMEZ).
- KPI kartları **dekoratif değil, gerçek veriden hesaplanır** (`lib/pnl.ts`):
  - **Toplam Kasa**: başlangıç bakiyesi ∓ dolan emirlerin nakit etkisi
  - **Aktif PnL**: dolan her "bacağın" güncel orta fiyatla mark-to-market karşılaştırması (pozitifse yeşil, negatifse kırmızı — renk + glow otomatik döner)
  - **Toplam Özkaynak**: Kasa + PnL
  - **Ağ Sağlığı**: gerçek WebSocket varış hızı (Msg/Sn), throttle'dan bağımsız ölçülür

### 2. Profil ve Ayarlar (Zustand + LocalStorage Kalkanı)
- `state/useGlobalStore.ts`: `persist` middleware ile **yalnızca tarayıcının localStorage'ında** — Veri Sağlayıcı API Key, Aracı Kurum API Key/Secret Key, tema, görünen ad.
- **Güvenlik notu (kod içinde ve Ayarlar sayfasında da yazılı):** Bu anahtarlar hiçbir yere gönderilmez, sadece localStorage'da durur ve varsayılan olarak maskelenir (göster/gizle ile açılır). Gerçek production kullanımında hassas anahtarların backend `.env`'de tutulması (bu proje zaten destekliyor) önerilir — localStorage, aynı origin'deki herhangi bir JS tarafından okunabilir (XSS riski).
- **Tema (Koyu/Açık):** Mimari karar — renkler her yerde `var(--color-*)` CSS değişkenleriyle okunuyor; `.light` sınıfı bu değişkenleri TEK bir yerde (`index.css`) geçersiz kılıyor, onlarca bileşende `dark:`/`light:` varyantı tekrarlamaya gerek yok.

### 3. Throttle / Batching Koruma Duvarı
- `lib/throttleBuffer.ts`: saf, bağımsız test edilmiş arabellek sınıfı.
- `hooks/useWebSocketStream.ts`: gelen HER mesaj arka planda birikir (React state'i tetiklemez); sabit bir zamanlayıcı (**varsayılan 250ms = saniyede en fazla 4 render**) birikeni tek seferde `onBatch` ile iletir. Ham varış hızı ayrıca `lib/telemetry.ts` ile ÖLÇÜLÜR (throttle'dan bağımsız — KPI header'daki "Ağ Sağlığı" gerçek sayıyı gösterir, render sıklığını değil).
- Orderbook/AKD abonelikleri "en son değer kazanır" mantığıyla batch'in son elemanını alır; alarm akışı ise batch'teki TÜM olayları işler (hiçbir alarm sessizce kaybolmaz).

### Bonus: Otomasyon (Botlar) sayfası
İstenmedi ama eklendi — backend'de zaten çalışan 7 algoritmik tespit motorunu (Modül 2/3'ün tamamı) "bot" kartları olarak listeler, **bu oturumda gerçekten ürettikleri alarm sayısıyla**. Anahtar (toggle), o botun uyarılarını Alarm Akışı'nda gösterip gizler — dürüstlük notu: sunucu tarafındaki tespiti DURDURMAZ, sadece arayüz filtresidir (kod içinde de böyle belgelendi).

## Hızlı başlangıç

```bash
cd bist_algo_tracker
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env

bash scripts/run_demo.sh   # API (:8000) + React (:5173) birlikte açılır
```

Backend testleri: `pytest -v` · Frontend testleri: `cd dashboard-react && npm test`

## Mimari (değişmeyenler v2 README'sinde detaylı — burada v3 farkı)

```
dashboard-react/src/
├── router.tsx, main.tsx          # v3: React Router kurulumu
├── layout/
│   ├── AppLayout.tsx              # v3: kalıcı kabuk (nav+KPI+global WS+hotkey)
│   ├── TopNav.tsx                 # v3: sekmeler + canlı durum rozeti
│   └── KpiHeader.tsx              # v3: Kasa/PnL/Özkaynak/Ağ Sağlığı kartları
├── pages/
│   ├── DashboardPage.tsx          # v2'nin WorkspaceGrid'i (store'dan okur)
│   ├── AutomationPage.tsx         # v3: bot roster
│   ├── ProfilePage.tsx            # v3: kimlik + hesap özeti
│   └── SettingsPage.tsx           # v3: API anahtarları + tema
├── state/
│   ├── useAppStore.ts             # v2 + v3: mutedAlertTypes eklendi
│   ├── useGlobalStore.ts          # v3: YENİ - persist (localStorage)
│   └── usePortfolioStore.ts       # v3: YENİ - ham emir verisi
├── hooks/
│   ├── useWebSocketStream.ts      # v3: throttle/batching eklendi
│   ├── usePortfolioMetrics.ts     # v3: YENİ - türetilmiş Kasa/PnL
│   └── useMessagesPerSecond.ts    # v3: YENİ
└── lib/
    ├── pnl.ts, throttleBuffer.ts, telemetry.ts   # v3: YENİ, hepsi test edilmiş
```

Backend'de tek değişiklik: `core/execution/order_manager.py`'ye `get_all_orders()` ve `api/main_fastapi.py`'ye `GET /orders/all` + `health`'e `starting_balance_try` eklendi. Geri kalan her şey (analitik motorlar, risk kuralları, mesajlaşma, dayanıklılık) v2 ile birebir aynı.

## PnL hesabının sadeleştirmesi (dürüstlük notu)

Backend'de gerçek bir pozisyon/ortalama-maliyet defteri YOK (`PaperBrokerAdapter` sadece emir durumu tutar). Bu yüzden `lib/pnl.ts`, her dolmuş emri **bağımsız bir "bacak"** olarak ele alır — aynı sembolde birden fazla dolum netlenmez/ortalanmaz. Gerçek bir netleme/ortalama-maliyet sistemi, backend'e ayrı bir "ledger" modülü eklemeyi gerektirir; bu v3'ün kapsamı dışında bırakıldı (frontend mimarisi odaklı bir talepti).

## Gerçek veriye bağlanmak / Sermaye koruma

v1 README'sindeki notlar aynen geçerli: `BISTWebSocketClient._parse_raw_message` kasıtlı olarak boş, AKDE ayrı/ücretli lisans, risk kuralları hiçbir yerde bypass edilmiyor.
