# BIST-AlgoTracker v3.5

Borsa İstanbul için algoritmik ticaret ve kurumsal veri analiz platformu. v3.5, v1→v3'ün backend'ini **bozmadan** üzerine 3 kritik modül ekler: **AppSec kimlik doğrulama katmanı**, **look-ahead-bias'a karşı korumalı backtest motoru** ve **çok katmanlı güvenlik kapılı otonom karar motoru**. Frontend (React trader terminali) v3'te tamamlandığı için bu turda değiştirilmedi.

## Neyin gerçekten doğrulandığı (iddia değil, bu ortamda test edildi)

✅ **149 pytest testi, hepsi geçti** (v1-v3'ün 143'ü + v3.5'in 64 yeni testinden **~58'i backend'e**, gerisi):
- **AppSec (Görev 1):** `SecretStr` maskelemesi f-string/repr üzerinden test edildi; log redaksiyon filtresi **gerçek bir logging.LogRecord üzerinde** çalıştırılıp ham secret'ın asla çıktıda görünmediği kanıtlandı; **iki eşzamanlı `asyncio.Task`'ın birbirinin kayıtlı secret'ını GÖRMEDİĞİ** (context izolasyonu) açıkça test edildi.
- **Backtest (Görev 2):** `Simulator`'ın look-ahead-bias koruması, "20 olaylık akışın HER adımında handler'ın gördüğü veri kümesinin TAM OLARAK o ana kadarki veri olduğunu" doğrulayan bir testle **kanıtlandı** (bkz. `test_handler_never_sees_future_data_before_its_time`); geriye sıçrayan zaman damgası `LookAheadViolationError` fırlatıyor.
- **Otonom Motor (Görev 3):** Her güvenlik kapısı (kill-switch, günlük zarar limiti, hız sınırlayıcı, risk motoru, fiyat yokluğu) **İZOLE OLARAK**, "güçlü sinyal olsa BİLE emir gönderilMEDİĞİ" + "brokera GERÇEKTEN hiçbir emrin gitmediği" (bağımsız `get_open_orders()` kontrolüyle) kanıtlanarak test edildi.
- Bandit: 0 bulgu.

✅ **Canlı sunucuda uçtan uca doğrulandı:**
- `GET /broker/status` header'sız → 401; header'lı → sadece boolean yanıt; **sunucu log dosyasında ham secret değerinin GEÇMEDİĞİ `grep` ile doğrulandı**.
- `AUTONOMOUS_TRADING_ENABLED=true` ile gerçekten çalıştırıldı: motor canlı sentetik veriyle karar üretti, ısınma sonrası **gerçek otonom emirler `/orders/all`'da göründü ve bir kısmı otomatik doldu** (aynı v3'teki auto-fill mekanizmasıyla).
- Varsayılan (`false`) durumda **birim testle** (canlı 25sn beklemek yerine — sandbox zaman kısıtı, aşağıda not) kill-switch'in en güçlü sinyalde bile hiçbir emir üretmediği kanıtlandı.

⚠️ **Doğrulanamadı / kapsam dışı (dürüstlük notu):**
- OHLCV bar'ın **canlı sunucuda gerçek bir dakika sınırını geçip DB'ye yazıldığı** (60+ saniyelik gerçek zaman bekleyen bir canlı test) bu oturumda sandbox'ın uzun-süreli arka plan süreç kısıtları nedeniyle tamamlanamadı — bunun yerine `BarAggregator` + `SQLiteTimeseriesStore` **birim testleriyle** (pencere kapanışı, OHLCV matematiği, yazma/okuma) ayrı ayrı kanıtlandı ve entegrasyon kodu kısa canlı çalıştırmada hatasız calisti (sunucu cökmedi).
- Gerçek Kafka/ClickHouse/K8s, gerçek aracı kurum entegrasyonu, tarayıcı görsel testi — v1'den beri bilinçli olarak kapsam dışı (ağ izin listesi/mimari kararlar - bkz. eski README notları).

## v3.5'te ne eklendi

### Görev 1 — AppSec Kalkanı (`api/dependencies/auth.py`, `config/log_redaction.py`)
- `BrokerCredentials`: `X-Broker-Key`, `X-Broker-Secret`, `X-Data-Provider-Key` header'larını `pydantic.SecretStr` olarak taşır — **repr()/str() HER ZAMAN maskelenir**, gerçek değere sadece `.get_secret_value()` ile ulaşılır (disipline değil tip sistemine dayanan garanti).
- **İkinci savunma hattı:** `SecretRedactingFilter`, kök logger'a takılıdır; bir istek sırasında kaydedilen (`register_secret_for_redaction`) ham değer HERHANGİ bir log kaydında (biri yanlışlıkla loglasa bile) geçerse `***REDACTED***` ile değiştirilir. `contextvars` ile istekler arası **izole** (bir isteğin secret'ı başka bir isteğin loglarını etkilemez — test edildi).
- `require_broker_credentials`: zorunlu kimlik bilgisi olmayan istekte 401 döner. `GET /broker/status` bunu canlı gösterir — **gerçek bir aracı kurum API'sine bağlanmaz** (o entegrasyon hâlâ bilinçli olarak yazılmadı).

### Görev 2 — Backtest ve Simülasyon Motoru (`core/backtest/`)
- `Simulator`: olayları KESİNLİKLE artan zaman damgası sırasında besler; geriye sıçrama `LookAheadViolationError` fırlatır (sessizce yeniden sıralamaz — bu, veri hazırlama hatasını gizlerdi).
- `AnalyticsEngineHandler`: v1/v2'nin GERÇEK analitik motorlarını (Spoofing/Layering/Iceberg/Institutional) Simulator'a bağlar — backtest ve canlı sistem AYNI tespit mantığını çalıştırır.
- `core/analytics/bar_aggregator.py` + `db/timeseries_client.py` (genişletildi): canlı EXECUTED akışından OHLCV bar üretip SQLite'a yazar — Simulator'ın "geçmiş OHLCV" kaynağı budur.

### Görev 3 — Otonom Tetikleyici (`core/analytics/feature_extractor.py`, `core/execution/autonomous_engine.py`)
- `FeatureExtractor`: OBI, sembole-özel z-score (adaptif eşik) ve hacim ivmesini standartlaştırılmış bir `FeatureVector`'e indirger — **karar VERMEZ**, sadece sinyal üretir.
- `AutonomousDecisionEngine`: sinyalden emre giden boru hattı. **TÜM** kapılardan geçmeden emir gönderilmez:
  1. `settings.autonomous_trading_enabled` — **varsayılan `False`** (kill-switch)
  2. `DailyLossGuard` — basit GÜNLÜK spot PnL limiti (Nema/taşıma maliyeti YOK — Kırmızı Çizgi #1)
  3. `OrderRateLimiter` — dakikada azami emir (emir-spam koruması)
  4. `RiskEngine.can_open_position` — mevcut %20 pozisyon limiti, **GÜNCEL** bakiye/maruziyetle (`core/execution/pnl.py` — basit spot, frontend'deki `lib/pnl.ts` ile aynı kural)
- `GET /autonomous/status`: motorun canlı durumunu (enabled, günlük PnL, son 20 karar ve NEDENİ) gösterir.
- **`_processing_loop`'a KAPALI olarak kablolandı** — `.env`'de `AUTONOMOUS_TRADING_ENABLED=true` yapmadan sistemin mevcut davranışı hiç değişmez.

## Hızlı başlangıç

```bash
cd bist_algo_tracker
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env       # AUTONOMOUS_TRADING_ENABLED=false varsayılan

bash scripts/run_demo.sh   # API (:8000) + React (:5173)
pytest -v                  # backend testleri
```

Otonom motoru denemek için (**sadece paper broker'a karşı, gerçek para YOK**): `.env`'de `AUTONOMOUS_TRADING_ENABLED=true` yapıp API'yi yeniden başlatın, `GET /autonomous/status` ile izleyin.

## Önceki sürümler

v1 (9 modül, temel platform), v2 (mesajlaşma/dayanıklılık/gelişmiş analitik), v3 (React trader terminali) hakkında detaylı bilgi bu depoda korunan `SYSTEM_PROMPT_BIST_AlgoTracker_v2.md` dosyasında ve kod içi docstring'lerde mevcuttur. Mimari özet: `BISTWebSocketClient._parse_raw_message` kasıtlı olarak boş (gerçek veri sağlayıcı bağlanmadı), AKDE ayrı/ücretli lisans gerektirir, tüm risk kuralları hiçbir yerde bypass edilemez.
