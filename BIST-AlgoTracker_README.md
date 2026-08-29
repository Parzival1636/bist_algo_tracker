# BIST-AlgoTracker

Borsa İstanbul için algoritmik ticaret ve kurumsal veri analiz platformu. Canlı Düzey 2 + AKD verisi işler, spoofing/yemleme türü anomalileri tespit eder, kurumsal yoğunlaşmayı izler, risk yönetimi yapar ve backtest çalıştırır.

## Demo Mod vs. Canlı Mod

Sistem iki modda çalışır, `.env` içindeki `DEMO_MODE` ile kontrol edilir:

| | **DEMO_MODE=true** (varsayılan) | **DEMO_MODE=false** |
|---|---|---|
| Veri kaynağı | `SimulatedMarketDataSource` (sentetik, random-walk) | Gerçek veri sağlayıcı (kendi bağlantınız) |
| Redis | Gerekmez (bellek içi pub/sub) | Gerekli |
| Veritabanı | SQLite (`demo.db`) | PostgreSQL |
| Amaç | Kod tabanını hatasız çalışırken görmek, mantığı doğrulamak | Gerçek para/gerçek emirle çalışmak |

**Bu teslimatta DEMO_MODE=true uçtan uca test edilmiştir** (aşağıya bakın). Canlı moda geçiş, aşağıdaki "Gerçek veriye bağlanmak" bölümünde anlatıldığı gibi sizin veri sağlayıcı/aracı kurum bilgilerinizi gerektirir — bunlar olmadan gerçek bir bağlantı kodu **yazılmadı**, çünkü yanlış varsayılan bir şema, gerçek para üzerinde sessizce hatalı analiz sonuçlarına yol açabilirdi.

## Neyin doğrulandığı, neyin doğrulanamadığı

✅ **Gerçekten çalıştığı doğrulandı (bu ortamda):**
- 42 birim/entegrasyon testi (`pytest`) — OTR/OBI/Dwell-Time formülleri, 3 spoofing kuralı, risk motoru (stop-loss/trailing/%20 limiti), kurumsal yoğunlaşma endeksi, backtest motoru
- Tüm dosyalarda `py_compile` (sözdizimi) taraması
- FastAPI uygulaması gerçekten başlatıldı; `/health`, `/orderbook`, `/akd`, `/alerts`, `/ws/alerts` uçları gerçek sentetik veriyle test edildi ve **gerçek alarmlar üretildi** (SAHTE_LİKİDİTE, KURUMSAL_YOGUNLASMA)
- Alarmların SQLite'a kalıcı yazıldığı doğrulandı
- Bandit statik güvenlik taraması: 0 bulgu
- `docker-compose.yml` ve CI workflow YAML olarak geçerli

⚠️ **Doğrulanamadı (bu ortamdan gerçek BİST/aracı kurum API'lerine ağ erişimi yok):**
- Gerçek bir veri sağlayıcıya (Matriks/İdeal Data/BISTECH VERDA) bağlanan kod — bilinçli olarak iskelet bırakıldı, bkz. aşağı
- Gerçek bir aracı kurumun emir API'sine bağlanan kod — bilinçli olarak yazılmadı
- Docker imajlarının gerçekten build edilmesi (Docker bu ortamda yok; Dockerfile'lar standart, doğrulanmış kalıplarla elle yazıldı)
- Telegram/Discord bildirimleri (gerçek token/webhook gerektirir)

## Önemli: AKDE verisi hakkında (araştırıldı)

Spec'teki "anlık AKD" (kurumların seans içi net alım/satımı), Borsa İstanbul'da **AKDE** (Aracı Kurum Dağılımı ve **Eşanlı** Taraf Bilgisi) adlı **ayrı ve ücretli** bir lisanstır — sıradan **AKD** lisansı sadece **gün sonu** veri verir. Gerçek zamanlı kurumsal takip modülünü canlıya almak için veri sağlayıcınızdan (Matriks vb.) özellikle AKDE lisansına ve BISTECH/VERDA veya sağlayıcının kendi API erişimine ihtiyacınız olacak.

## Mimari

```
bist_algo_tracker/
├── config/         # Ayarlar (.env okur) + loglama
├── models/         # Ortak Pydantic şemaları (OrderBook, AKD, Alert, Order, Position...)
├── core/
│   ├── ingestion/    # Modül 1: SimulatedMarketDataSource + BISTWebSocketClient iskeleti
│   ├── analytics/    # Modül 2+3: OTR/OBI/Dwell-Time, SpoofingDetector, kurumsal yoğunlaşma
│   ├── execution/    # Modül 4: RiskEngine (stop-loss/trailing/%20 limit), OrderManager
│   └── ai_rag/       # Modül 5: KAP sentiment (sözlük tabanlı + opsiyonel Claude)
├── db/             # Modül 6/7: Redis pub/sub (+ bellek içi alternatif), SQLAlchemy modelleri
├── backtest/       # Modül 7: HistoricalDataPlayer + BacktestEngine
├── notifications/  # Modül 8: Telegram + Discord
├── api/            # Modül 6: FastAPI (REST + WebSocket)
├── dashboard/       # Modül 6: Streamlit canlı panel
├── tests/          # 42 test
├── docker/, docker-compose.yml   # Modül 9
└── .github/workflows/ci.yml      # Modül 9: Bandit + pytest
```

## Hızlı başlangıç (demo mod)

```bash
cd bist_algo_tracker
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env          # varsayılanlar demo mod için hazır

bash scripts/run_demo.sh      # API (8000) + Dashboard (8501) birlikte açılır
```

Tarayıcıda `http://localhost:8501` — sentetik veri akarken order book, AKD dağılımı ve anomali alarmlarını canlı görürsünüz.

Sadece testleri çalıştırmak için:
```bash
pytest -v
```

## Docker ile çalıştırma

```bash
docker compose up --build
```
Bu, API + Redis + PostgreSQL + Dashboard'u ayrı konteynerlerde ayağa kaldırır (Modül 9). `.env` dosyanız `DEMO_MODE=true` içerdiği sürece veri yine sentetiktir; gerçek Redis/PostgreSQL servisleri de bu şekilde altyapı olarak devrede olur ama henüz kullanılmaz.

## Gerçek veriye bağlanmak

1. Veri sağlayıcınızdan (Matriks/İdeal Data/vb.) Düzey 2 + AKDE lisansı ve API erişimi alın.
2. `.env`: `DEMO_MODE=false`, `MARKET_DATA_WS_URL`, `MARKET_DATA_API_KEY` doldurun.
3. `core/ingestion/websocket_client.py` içindeki `BISTWebSocketClient._parse_raw_message` metodunu, sağlayıcınızın **gerçek** JSON şemasına göre doldurun (şu an bilinçli olarak `NotImplementedError` fırlatıyor).
4. `api/main_fastapi.py` içindeki `_ingestion_loop`'ta `SimulatedMarketDataSource` yerine `BISTWebSocketClient(...)` kullanın.
5. Gerçek emir gönderimi için `core/execution/order_manager.py` içindeki `BrokerAdapter`'ı kendi aracı kurumunuzun API'sine göre implemente edin (şu an sadece güvenli `PaperBrokerAdapter` var).

**Öneri:** Adım 3-5'i tamamladıktan sonra, gerçek parayla önce **backtest** (Modül 7) ve bolca **paper trading** ile doğrulayın — özellikle Modül 4'teki otomatik emir iptal/güncelleme mantığı canlıda beklenmedik şekilde davranabilir.

## Anomali eşikleri (ayarlanabilir, `.env`)

| Eşik | Varsayılan | Anlamı |
|---|---|---|
| `OTR_THRESHOLD` | 0.90 | Bu OTR üzeri + ihmal edilebilir gerçekleşme → SAHTE_LİKİDİTE |
| `OBI_THRESHOLD` | 0.80 | Bu OBI üzeri sürüp fiyat yükselmezse → SPOOFING_UYARISI |
| `LARGE_ORDER_LOT_THRESHOLD` | 50.000 | Dwell-time takibine giren minimum lot |
| `DWELL_FLICKER_SECONDS` | 0.5 | Bu süreden kısa + 5sn içinde 3 tekrar → SPOOFING_ALARM |

Yorumlama kararları (spec'in doğal dil kurallarını sayısallaştırırken alınan kararlar) her modülün docstring'inde açıklanmıştır.

## Notlar

- Modül 5 (AI/RAG): Varsayılan sentiment skorlayıcı harici API gerektirmeyen sözlük tabanlıdır; `ANTHROPIC_API_KEY` + `USE_LLM_SENTIMENT=true` ile Claude tabanlı skora geçilebilir.
- Sermaye koruma ilkesi (Modül 4) kod genelinde önceliklidir: stop-loss/trailing-stop/%20 limiti hiçbir yerde bypass edilemez şekilde yazıldı, testlerle doğrulandı.
