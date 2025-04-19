
# Major FX Pairs Historical Minute‑Level Data Sources  
*(Last updated: 2025-04-19)*

This report catalogs free and paid repositories that distribute historical **1‑minute (or finer)** OHLC/quote data for major FX pairs (e.g., *EUR/USD, USD/JPY, GBP/USD*).  
Focus is on data that can ultimately be loaded into **MariaDB** for systematic research and back‑testing.

---

## 1. Free Data Sources

| Provider | Coverage (≈M1) | Format & Access | Cost / Procedure | Notes & Licence |
|----------|----------------|-----------------|------------------|-----------------|
| **HistData.com** | ~2000 → Today | Monthly CSV (M1 OHLC/Raw tick) | Free download, no login; *$27* once‑off FTP optional | Widely used, minor gaps possible, licence permissive |
| **Dukascopy** | 2003 → Today | CSV via web tool or JForex; tick → any TF | Free (requires demo account) | Bank‑grade quotes; **personal use only** |
| **TrueFX** | 2009 → Today (recent 12‑mo free) | Monthly ZIP, tick Bid/Ask | Free sign‑up | Inter‑bank indicative; long‑term archive gated |
| **Gain Capital (legacy)** | 2001 → 2019 | Monthly ZIP tick CSV | Free (site archived) | Long range but gaps; project discontinued |
| **FXDD (legacy)** | 2005 → 2014 | Monthly ZIP `.hst` (MT4) ⇒ CSV export | Free (Wayback / MT4 history) | Support ended; licence unclear; use as *supplement* |
| **OANDA API** | 2005 → Today | REST JSON/CSV (granularity=*M1..S5*) | Free demo token; rate‑limits | Broker quotes; no redistribution |
| **Other MT4 history** | Varies (broker‑dependent) | MT4 *History Center* pull | Free with demo | Requires CSV export; coverage limited |

> **TIP – “download mt4 historical data”** resources mostly re‑describe the MT4 *History Center* workflow and do **not** introduce new public datasets. Good background, but the main long‑range sources remain those listed above.

### 1.1 FXDD (legacy free M1 data)  
*Formerly hosted at* `global.fxdd.com/.../mt1m-data.html` *(site closed)*  

* **Format & Pairs** – `.hst` (MetaTrader native) for 20+ major pairs; convert to CSV inside MT4.  
* **Period** – mainly **2005 – 2014 H1**, pair‑dependent.  
* **Retrieval today** –  
  1. Wayback Machine ZIPs 2. FXDD MT4 demo ⇢ *History Center* 3. Community mirrors/GitHub.  
* **Quality** – Minor gaps/spikes; 2005 earliest.  
* **Licence** – Free but no official redistribution clause → treat as *self‑use only*.  

Sample MariaDB load SQL (CSV with `YYYY.MM.DD,HH:MM,Open,High,Low,Close,Vol`):

```sql
CREATE TABLE eurusd_m1 (
  ts      DATETIME PRIMARY KEY,
  o DOUBLE, h DOUBLE, l DOUBLE, c DOUBLE, v INT
);
LOAD DATA LOCAL INFILE '/data/EURUSD_all.csv'
INTO TABLE eurusd_m1
FIELDS TERMINATED BY ',' IGNORE 1 LINES
(@d,@t,o,h,l,c,v)
SET ts = STR_TO_DATE(CONCAT(@d,' ',@t),'%Y.%m.%d %H:%i');
```

---

## 2. Paid Data Sources

| Provider | Coverage | Format & Access | Price (indicative) | Commercial Use |
|----------|----------|-----------------|--------------------|----------------|
| **TraderMade** | ≈20 yrs+ | REST JSON/CSV, GUI bulk | Free ≤1 k req/mo; £30 +/mo plans | ✔ (licensed) |
| **Tick Data LLC** | 1990s → Today | ASCII CSV; tooling | Dataset buy‑out – *$k* | ✔ (full licence) |
| **Kibot** | 2009 → Today | CSV/TXT bundles | Few $100s per pair‑set | ✔ (in‑house) |
| **Xignite / Polygon / Twelve Data** | 10–20 yrs (limited granularity) | REST/WS JSON | Subscriptions from $29 +/mo | ✔ |
| **Refinitiv / FactSet** | 30 yrs+ tick | Proprietary | Enterprise, to be quoted | ✔ |

---

## 3. Comparison & Recommendation

* **Longest free history** – Dukascopy (tick) ➜ aggregate to M1; HistData.com ~2000–Today.  
* **Easiest bulk free grab** – HistData.com CSV monthly zips.  
* **Highest fidelity** – Tick Data LLC (paid) or Dukascopy tick.  
* **Licence‑clean for commercial** – TraderMade Basic/Pro, Tick Data, Kibot.  
* **Legacy supplement** – FXDD & GainCapital archives fill 2000s gaps but need cleansing.  

---

## 4. ETL into MariaDB – Best‑practice Checklist

1. **Unify time‑zone** to UTC before load.  
2. **De‑duplicate** overlaps: prefer highest‑quality source as primary, others only for missing bars.  
3. **Validate continuity** – detect ≥2× interval gaps; forward‑fill or flag.  
4. **Partition tables** by year or month to speed range queries (`PARTITION BY RANGE (YEAR(ts))`).  
5. **Compress** with `ROW_FORMAT=COMPRESSED` if using InnoDB & large datasets.  
6. **Document licence & provenance** per table for future audits.

---

## 5. References & Further Reading

* HistData.com – *Free Forex Historical Data*  
* Dukascopy **Historical Data Feed** documentation  
* TrueFX – *Tick-by-Tick Market Data*  
* FXDD Wayback snapshot *(2014‑02‑14)*  
* TradingHeroes – *How to Download MT4 Historical Data* (2022)  
* TickData LLC – *FX Tick Database*  

*(URLs omitted here; see browser history citations in conversation for source verification.)*
