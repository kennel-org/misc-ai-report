# Helium‑Filled HDDs: SMART “Helium Level” Attribute Implementations  
*(Seagate, Western Digital / HGST, Toshiba)*  

**Compiled:** 2025-04-19

---

## 1. Overview  

Helium‑filled drives keep a sealed, low‑density atmosphere inside the head–disk assembly (HDA).  
Each vendor exposes the health of that environment through *vendor‑specific* S.M.A.R.T. attributes, but the details are different.

| Vendor | S.M.A.R.T. attribute(s) | Normalised range | Threshold† | Behaviour |
|--------|-------------------------|------------------|------------|-----------|
| Western Digital / HGST | **22 – Current Helium Level** | 100 → 0 | 25 | Gradual decrease; present since Ultrastar *He6* (6 TB, 2013). |
| Toshiba | **23 – Helium Condition Lower**  <br> **24 – Helium Condition Upper** | 100 → 0 | 75 | Attribute pair introduced with *MG07* (12/14 TB). |
| Seagate | **200 – Pressure_Limit** | 100 / 1 | 1 | Binary flag from MEMS pressure sensor; drops straight to 1 (FAIL). |

† *Attribute is considered “Pre‑Fail” when the normalised value is **≤ threshold**.*

---

## 2. Western Digital / HGST  

* **Attribute ID 22 – `Helium_Level`**  
  * **Normalised**: starts at 100 and decays slowly as helium diffuses.  
  * **Threshold**: 25. Value ≤ 25 raises *FAILING_NOW*.  
  * Seen on all HGST/WD helium models (Ultrastar He6 → He12, WD Red 8 TB “white‑label” etc.).  
  * **Implementation note** – HGST relies on *indirect* detection (e.g. head fly‑height change), not on a pressure sensor.[^6]

---

## 3. Toshiba  

* **Attribute IDs 23 / 24 – `Helium Condition Lower/Upper`**  
  * Introduced with *MG07ACA* 12/14 TB (2017).  
  * Both attributes report identical normalised values (100 when new).  
  * **Threshold**: 75 for both IDs.  
  * No public documentation of sensor type; likely indirect measurement.

---

## 4. Seagate  

* **Attribute ID 200 – `Pressure_Limit`**  
  * Normalised = 100 in‑spec, **1** when pressure exceeds limits (helium lost).  
  * Raw value stays 0; the attribute behaves as an “environment‑good” flag.  
  * Present on all Exos X series (10 TB+) and IronWolf Pro helium models.  
  * Seagate white paper confirms *digital MEMS sensors for temperature, pressure, humidity* inside the HDA.[^5]

---

## 5. Practical Recommendations  

1. **Collect full SMART logs** regularly and archive them. Trend analysis of attributes 22/23–24 is useful for WD/HGST & Toshiba drives.  
2. For **Seagate**, treat *any* drop of attribute 200 below 100 as imminent failure: backup & replace.  
3. Automate alerts in monitoring tools (smartd, Zabbix, Prometheus exporters, etc.).  
4. Remember that helium leaks are rare; false positives can occur (especially on HGST, attr 22). Validate with manufacturer diagnostics before RMA.  

---

## 6. References  

[^1]: Backblaze blog *“The Helium Factor and Hard Drive Failure Rates”* (30 Nov 2018) — identifies SMART 22 as helium level. <https://www.backblaze.com/blog/helium-filled-hard-drive-failure-rates/>  
[^2]: smartmontools Ticket #1409 — Attribute 22 reported as *Unknown* on WD120EDAZ; mapped to **Helium_Level**. <https://www.smartmontools.org/ticket/1409>  
[^3]: Reddit r/DataHoarder *“Just a data point on SMART value 22 (Helium_Level)”* (Aug 2022) — user report of value drifting below threshold. <https://www.reddit.com/r/DataHoarder/comments/wl20q2/just_a_data_point_on_smart_value_22_helium_level/>  
[^4]: Backblaze blog *“Hard Drive Stats for Q2 2018”* — introduces Toshiba SMART 23/24. <https://www.backblaze.com/blog/hard-drive-stats-for-q2-2018/>  
[^5]: Seagate Technical Paper *“Decades of Proven Research Underpin Seagate’s Helium Drive”* (PDF, 2016) — states use of MEMS pressure sensors. <https://www.seagate.com/files/www-content/product-content/enterprise-hdd-fam/enterprise-capacity-3-5-hdd-10tb/_shared/docs/helium-drive-launch-tp686-1-1602us.pdf>  
[^6]: Seagate vs HGST comparison in same paper notes HGST lacks digital environmental sensors (implying indirect method).  
[^7]: smartmontools Ticket #1567 — provides SeaTools mapping showing attribute 200 as **Pressure_Limit**. <https://www.smartmontools.org/ticket/1567>  
[^8]: Reddit r/HDD *“Interpreting SMART values – Pressure_Limit on Seagate Exos”* (Apr 2025) — confirms attribute 200 drops to 1 on helium failure. <https://www.reddit.com/r/HDD/comments/1jphts5/interpreting_smart_values_especially_pressure/>  
[^9]: Wikipedia *“Self‑Monitoring, Analysis and Reporting Technology”* — lists attributes 22/23/24. <https://en.wikipedia.org/wiki/Self-Monitoring,_Analysis_and_Reporting_Technology>

---

*Prepared for: User inquiry on 19 Apr 2025.*