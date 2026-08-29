# BIST-AlgoTracker v2

Borsa İstanbul için algoritmik ticaret ve kurumsal veri analiz platformu. v2, v1'in 9 modülünü (aynı iş mantığıyla) korur; üzerine **performans/dayanıklılık katmanı** (Kafka-benzeri mesajlaşma, worker sharding, circuit breaker, backpressure) ve **gerçek bir trader terminali** (React DOM ladder, click-to-trade, hotkey'ler, sesli alarm) ekler.

Bu belge, `SYSTEM_PROMPT_BIST_AlgoTracker_v2.md` şartnamesinin **uygulanmış halidir**.

## Neyin gerçekten doğrulandığı (iddia değil, bu ortamda test edildi)

✅ **Backend — 83 pytest testi, hepsi geçti:**
- v1: OTR/OBI/Dwell-Time formülleri, 3 spoofing kuralı, risk motoru, kurumsal yoğunlaşma, backtest
- v2: Layering, Iceberg, adaptif (z-score) eşikler, çapraz-sembol korelasyon, TWAP/VWAP planlama, circuit breaker, backpressure kuyruğu, zaman serisi deposu, çoklu-process sembol worker'ı
- **Gerçek bir `nats-server` ikili dosyası indirilip çalıştırıldı** ve `NATSMessageBus` ona karşı gerçek pub/sub ile test edildi (bkz. `tests/test_nats_messaging.py` — nats-server yoksa otomatik `skip` edilir, base test suite'i etkilemez)
- `/orders` uçları FastAPI `TestClient` ile place→list→cancel akışı olarak test edildi
- Canlı sunucu (`uvicorn`) gerçekten ayağa kaldırıldı; `/health`, `/orderbook`, `/akd`, `/alerts`, `/orders`, `/ws/alerts`, `/ws/orderbook/{symbol}`, `/ws/akd/{symbol}` hepsi gerçek sentetik veriyle test edildi ve **v2'nin 3 yeni alarm tipi de (LAYERING_ALARM, ICEBERG_TESPIT, CAPRAZ_SEMBOL_KORELASYON) canlı olarak üretildiği görüldü**
- Bandit: 0 bulgu

✅ **Frontend — 18 vitest testi, hepsi geçti:**
- `tsc -b` (strict mode, `noUnusedLocals`/`noUnusedParameters` dahil): 0 hata
- `vite build`: temiz production build (0 hata, 0 uyarı)
- `oxlint`: 0 hata, 0 uyarı
- `vite preview` ile üretilen statik dosyalar gerçekten servis edildi ve HTTP 200 doğrulandı
- Saf mantık (fiyat merdiveni, donut grafik yüzdeleri) vitest ile bağımsız test edildi

⚠️ **Doğrulanamadı (dürüstlük notu):**
- **Görsel/tarayıcı testi yapılamadı** — bu ortamda headless tarayıcı (Playwright/Chromium) indirilemiyor (`cdn.playwright.dev` ağ izin listesinde değil). Kod TypeScript seviyesinde tip-güvenli ve build ediliyor, ama gerçek DOM render'ını/tıklamaları gözle görmedim. İlk çalıştırmada küçük CSS/layout düzeltmeleri gerekebilir.
- Gerçek Kafka/ClickHouse/Kubernetes kurulup test edilmedi — bkz. aşağıdaki "Neden NATS/SQLite" notu.
- Gerçek aracı kurum/veri sağlayıcı API'si (v1'den beri) hâlâ bağlanmadı — bilinçli olarak.

## Hızlı başlangıç

```bash
cd bist_algo_tracker
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env

bash scripts/run_demo.sh
```
Bu, API'yi (`:8000`) ve React trader arayüzünü (`:5173`, ilk çalıştırmada otomatik `npm install` yapar) birlikte açar.

Sadece backend testleri: `pytest -v` · Sadece frontend testleri: `cd dashboard-react && npm test`

Hafif Streamlit paneli (v1, hâlâ mevcut) için: `bash scripts/run_demo_streamlit.sh`

## Docker ile (tüm mikroservisler)

```bash
docker compose up --build
```
API (`:8000`), Redis (`:6379`), NATS (`:4222`), PostgreSQL (`:5432`), Streamlit (`:8501`) ve React (`:5173`) ayrı konteynerlerde ayağa kalkar.

## Mimari (v1 → v2)

```
bist_algo_tracker/
├── core/analytics/
│   ├── metrics.py, spoofing_detector.py, institutional.py   # v1
│   ├── adaptive_thresholds.py   # v2: z-score bazlı dinamik esik
│   ├── layering_detector.py     # v2: cok-kademeli spoofing
│   ├── iceberg_detector.py      # v2: buzdagi emir tespiti
│   └── correlation_engine.py    # v2: capraz sembol/kurum korelasyonu
├── core/execution/
│   ├── order_manager.py, risk_engine.py   # v1
│   └── algo_execution.py                  # v2: TWAP/VWAP planlama
├── core/resilience/            # v2: YENI
│   ├── circuit_breaker.py       # pybreaker sarmalayici
│   └── backpressure.py          # sinirli, "en eskiyi at" kuyruk
├── messaging/                  # v2: YENI - Redis Pub/Sub'in yerini alir
│   ├── bus.py, in_memory_bus.py, nats_bus.py, factory.py
├── workers/                    # v2: YENI
│   └── symbol_worker.py         # multiprocessing ile izole sembol analizi
├── db/
│   ├── redis_client.py, postgres_models.py   # v1 (redis_client artik kullanilmiyor - bkz. not)
│   └── timeseries_client.py                   # v2: YENI (ClickHouse/TimescaleDB arayuzu)
├── api/main_fastapi.py         # v1 + v2 (tum yeni dedektorler/uclar buraya kablolandi)
├── dashboard/app.py            # v1 Streamlit - fallback olarak KORUNDU
├── dashboard-react/            # v2: YENI - birincil trader arayuzu (asagida detay)
├── tests/                      # v1: 6 dosya, v2: +7 dosya (83 test toplam)
└── SYSTEM_PROMPT_BIST_AlgoTracker_v2.md   # bu uygulamanin kaynak sartnamesi
```

**Not (dürüstlük):** `db/redis_client.py` v1'den kalma; `api/main_fastapi.py` artık `messaging/factory.py` üzerinden `MessageBus`'ı kullanıyor (`MESSAGE_BUS=memory|nats`). Redis modülü, `MESSAGE_BUS` ne olursa olsun ondan tamamen bağımsız (gerçek Redis'e geçmek isterseniz `messaging/` altına bir `redis_bus.py` eklemek en doğal yol olur — kolayca yapılabilir ama bu iterasyonda NATS'ı önceliklendirdik çünkü gerçek bir sunucuya karşı test edebildik).

### dashboard-react/ — trader arayüzü

```
dashboard-react/src/
├── components/   DomLadder, OrderTicket, AlertFeed, AkdPanel, Watchlist,
│                 ChartPanel, OrdersPanel, WorkspaceGrid, TopBar, PanelShell
├── lib/          priceLadder.ts, donut.ts, format.ts, soundAlert.ts (+ testler)
├── hooks/        useWebSocketStream (oto-yeniden-baglanan), useTraderHotkeys
├── state/        useAppStore.ts (Zustand, selector-bazli)
└── api/client.ts REST + WS istemcisi
```

**Uygulanan trader-odaklı ozellikler (v2 spec Bolum 4):**
- Sürüklenebilir/yeniden boyutlandırılabilir panel düzeni (`react-grid-layout`)
- Fiyat-merkezli DOM ladder, hacme göre renk yoğunluğu, SAHTE_LİKİDİTE/şüpheli kademe vurgusu, **click-to-trade** (tıkla → emir formu dolar)
- Emirler gerçekten `PaperBrokerAdapter`'a gider ve piyasa fiyatı kesince **otomatik dolar** (bkz. `_try_fill_open_orders`) — hâlâ kağıt/simülasyon, gerçek para yok
- Klavye kısayolları: `F1` piyasa alış, `F2` piyasa satış, `Esc` tüm açık emirleri iptal
- Sesli alarm (Web Audio API, dosya gerekmez) — CRITICAL için çift bip
- TradingView Lightweight Charts (canlı orta-fiyat çizgisi — tam mum grafiği için not aşağıda)
- `react-window` ile sanallaştırılmış alarm akışı (DOM'da sadece görünen satırlar)
- Tüm REST uçları + **3 WebSocket kanalı** (`/ws/alerts`, `/ws/orderbook/{symbol}`, `/ws/akd/{symbol}`) — polling YOK

**Kapsam dışı bırakılanlar (dürüstlük):** Isı haritası (market-profile tarzı), tam OHLC mum grafiği (backend bar agregasyonu gerektirir), kullanıcı bazlı panel-düzeni kalıcılığı (backend endpoint'i yok), çoklu-pencere/monitör desteği. Bunlar v2 şartnamesinde vardı ama zaman/kapsam nedeniyle bu iterasyona alınmadı — mimari (WorkspaceGrid, MessageBus) bunları eklemeye açık.

## Neden NATS (Kafka değil) ve SQLite (ClickHouse değil)?

v2 şartnamesi Kafka/ClickHouse/Kubernetes öneriyordu. Bu ortamda:
- **NATS** kullanıldı: tek bir Go ikili dosyası (`nats-server`), GitHub Releases'ten indirilip gerçekten çalıştırılabildi — Kafka bir JVM + ZooKeeper/KRaft kurulumu gerektirir, bu sandboxta pratik değildi. NATS de aynı temel kazancı verir: kalıcı/replay edilebilir mesajlaşma, sembol-bazlı subject'ler.
- **SQLite** (`db/timeseries_client.py`): ClickHouse/TimescaleDB bu ortamda kurulamadı. Arayüz (`TimeseriesStore` ABC) production'da `ClickHouseTimeseriesStore` ile değiştirilebilecek şekilde tasarlandı.
- **Worker sharding**: Kubernetes yerine gerçek `multiprocessing.Process` ile AYNI izolasyon ilkesi tek makinede gösterildi/test edildi (`workers/symbol_worker.py`).

## Anomali eşikleri ve v2 yorumlama kararları

| Tespit | Esik/Parametre | Not |
|---|---|---|
| OTR (v1) | `OTR_THRESHOLD=0.90` | Sabit |
| OBI (v1) | `OBI_THRESHOLD=0.80` | Sabit |
| OBI (v2) | z-score, `min_samples=30`, `z=3.0` | Sabit eşiğe EK sinyal, sembole özel kalibre |
| Layering | 3 kademe, 0.75sn pencere, ≥20.000 lot | `core/analytics/layering_detector.py` |
| Iceberg | ≥4 tekrar, göreli std ≤%15 | `core/analytics/iceberg_detector.py` |
| Çapraz-sembol | ≥3 sembol, 300sn pencere | `core/analytics/correlation_engine.py` |

Tüm yorumlama kararlarının gerekçesi ilgili modülün docstring'inde.

## Gerçek veriye bağlanmak (v1'den değişmedi)

`core/ingestion/websocket_client.py` içindeki `BISTWebSocketClient._parse_raw_message` hâlâ kasıtlı olarak boş (`NotImplementedError`) — hangi sağlayıcıyı kullanacağınız bilinmiyor. Ayrıca **AKDE** (anlık AKD) BİST'te ayrı/ücretli bir lisanstır; sıradan AKD lisansı sadece gün sonu verir (v1'den beri değişmeyen önemli bir nokta).

## Sermaye koruma ilkesi

v1'deki risk kuralları (stop-loss/trailing/%20 limit) hiçbir yerde bypass edilmedi. Yeni `algo_execution.py` (TWAP/VWAP) da GERÇEK EMİR GÖNDERMEZ, sadece dilim planı hesaplar — gönderim hâlâ test edilmiş `RiskEngine`/`OrderManager` üzerinden. Gerçek parayla önce backtest + bolca paper trading ile doğrulayın.
