# Helium‑Filled HDDs: SMART “Helium Level” Attribute Implementations  
*(Seagate | Western Digital / HGST | Toshiba)*  

**Updated:** 2025-04-20 00:53:06 UTC+09:00

---

## 1. Quick Reference  

| Vendor | SMART ID | smartctl label | Normalised range | Threshold | Comment |
|--------|:-------:|---------------|------------------|-----------|---------|
| Western Digital / HGST | **22** | `Helium_Level` | 100 → 0 | 25 | Gradual decay since Ultrastar He6. |
| Toshiba | **23 & 24** | `Helium Condition Lower/Upper` | 100 → 0 | 75 | Pair of attributes, MG07+. |
| Seagate | **200** | `Pressure_Limit` | 100 ↘ 1 | 1 | Binary flag from MEMS pressure sensor. |

---

## 2. Sample SMART Excerpts  

Below are *publicly available* excerpts illustrating how each attribute appears in real logs.

### 2‑A  WD Red 8 TB (Helium, WD80EFZX)  
```text
22 Helium_Level            PO---K   100   100   025    -    100
```  
Source: Red Hat Bug 1672853 comment (smartctl‑x WD80EFZX log). citeturn4view0  

---

### 2‑B  Toshiba MG07ACA14TE (14 TB)  
```text
23 Unknown_Attribute       PO---K   100   100   075    -    0
24 Unknown_Attribute       PO---K   100   100   075    -    0
```  
Source: smartmontools Ticket #1023 comment 7. citeturn5view0  

---

### 2‑C  Seagate Exos X20 ST20000NM007D  
```text
-v 200,raw48,Pressure_Limit   # mapping in smartmontools drivedb
...
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE  RAW_VALUE
200 Pressure_Limit          -O---K   100   100   001    Pre-fail     0
```  
Source: smartmontools Ticket #1567 attachment & change‑history comment. citeturn6view0turn10view0  

*(Note: Exos drives report 100 when pressure is within spec; drops instantly to 1 when out‑of‑spec.)*

---

## 3. Primary Literature / Data Sources  

| # | Reference | Vendor | Notes |
|---|-----------|--------|-------|
| 1 | Backblaze Blog **“The Helium Factor and Hard Drive Failure Rates”** (2018‑11‑30) | WD/HGST | First public confirmation of attr 22 as Helium_Level. citeturn0search1 |
| 2 | smartmontools **Ticket #1409** – map attribute 22 | WD/HGST | Adds decoding to drivedb. |
| 3 | Backblaze **Q2 2018 Drive Stats** – intro of Toshiba attrs 23/24 | Toshiba | Cited in smartmontools Ticket #1023. |
| 4 | Seagate White Paper **TP686‑1** – MEMS sensors in helium drives | Seagate | Confirms pressure sensor hardware. |
| 5 | smartmontools **Ticket #1567** – attribute 200 = Pressure_Limit | Seagate | Provides sample logs & drivedb entry. citeturn3view0 |
| 6 | GitHub **openSeaChest Issue #108** – shows “Pressure_Limit” mapping | Seagate | Mirrors drivedb update. citeturn10view0 |
| 7 | Reddit **r/DataHoarder** – attr 22 drift case study | WD/HGST | Field observation of gradual value drop. |

---

## 4. Usage Guidance  

*See previous revision for detailed operational advice.*  

---

*Document regenerated including explicit data‑source excerpts as requested.*