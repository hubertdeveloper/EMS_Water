<div align="center">

# 💧 EMS Water

### System Zarządzania Wodą · *Energy Management System*

Kompletna aplikacja SCADA/EMS dla zużycia wody — zbudowana w **Ignition 8.3 (Perspective)**
na bazie **PostgreSQL**, z symulatorem 18 liczników, archiwizacją, agregacjami, alarmami i analizą.

<br/>

![Ignition](https://img.shields.io/badge/Ignition-8.3.4-EE3923?style=for-the-badge)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18.3-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Perspective](https://img.shields.io/badge/Perspective-UI-183E66?style=for-the-badge)
![Jython](https://img.shields.io/badge/Jython-skrypty-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Status](https://img.shields.io/badge/status-uko%C5%84czony-2E8B57?style=for-the-badge)

</div>

---

## 📑 Spis treści

- [✨ Funkcje](#-funkcje)
- [🏗️ Architektura](#️-architektura)
- [🗄️ Struktura bazy danych](#️-struktura-bazy-danych)
- [🖥️ Ekrany aplikacji](#️-ekrany-aplikacji)
- [🔐 Użytkownicy i role](#-użytkownicy-i-role)
- [⚙️ Backend](#️-backend)
- [👥 Autorzy](#-autorzy)

---

## ✨ Funkcje

- 💧 **18 liczników wody** (M01–M18) w **6 lokalizacjach** — dane na żywo z symulatora
- 🗄️ **Pełny model danych** w PostgreSQL: konfiguracja, pomiary, agregacje, alarmy
- ⏱️ **Archiwizacja** pomiarów co 10 s + **agregacje** dzienne / tygodniowe / miesięczne / roczne
- 📊 **Dashboardy i raporty** — karty KPI, wykresy trendów, zestawienia per licznik i lokalizacja
- 🔎 **Moduł analiz** — filtrowanie po lokalizacji i okresie, wykresy porównawcze
- 🖥️ **Tabele „pełny ekran"** jednym kliknięciem
- 🔐 **Bezpieczeństwo** — role, Security Levels, Security Zones, użytkownicy z bazy

---

## 🏗️ Architektura

```mermaid
flowchart LR
    SIM["💧 Symulator<br/>(plik CSV)"] -->|Device Connection| GW

    subgraph GW["⚙️ Ignition Gateway 8.3"]
        direction TB
        T["🏷️ Tagi + UDT licznika"]
        TS["⏱️ Timery<br/>10 s · 30 s · 60 s"]
        NQ["🧩 Named Queries"]
    end

    GW -->|zapis + agregacje| DB[("🗄️ PostgreSQL<br/>EMS_Water")]
    DB --> NQ
    NQ --> UI["🖥️ Perspective UI"]
    T --> UI
    UI --> USR(["👤 Operator · Engineer · Analyst · Admin"])
```

Symulator zasila tagi w Gatewayu → timery archiwizują pomiary i liczą agregacje do bazy →
Named Queries pobierają dane → interfejs **Perspective** prezentuje je użytkownikom wg ról.

---

## 🗄️ Struktura bazy danych

> Schemat encji bazy **EMS_Water** (PostgreSQL). Klucze `PK` / `FK` zaznaczone przy kolumnach.

```mermaid
erDiagram
    ems_w_locations  ||--o{ ems_w_meters              : "zawiera"
    ems_w_meters     ||--o{ ems_w_measurements        : "rejestruje"
    ems_w_meters     ||--o{ ems_w_daily_consumption   : "dzienne"
    ems_w_meters     ||--o{ ems_w_weekly_consumption  : "tygodniowe"
    ems_w_meters     ||--o{ ems_w_monthly_consumption : "miesięczne"
    ems_w_meters     ||--o{ ems_w_annual_consumption  : "roczne"

    ems_w_locations {
        serial   location_id   PK
        varchar  location_code
        varchar  location_name
        varchar  building
        int      floor
        bool     is_active
    }
    ems_w_meters {
        serial   meter_id          PK
        int      location_id       FK
        varchar  meter_code        "M01..M18"
        varchar  meter_name
        varchar  tag_path
        numeric  nominal_flow_m3_h
        numeric  max_flow_m3_h
        numeric  alarm_threshold
        varchar  manufacturer
        bool     is_active
    }
    ems_w_measurements {
        bigserial    measurement_id   PK
        int          meter_id         FK
        timestamptz  recorded_at
        numeric      total_volume_m3
        numeric      flow_m3_h
        numeric      pressure_bar
        numeric      temperature_c
        bool         valve_status
        bool         leak_suspected
        int          data_quality
    }
    ems_w_daily_consumption {
        serial   daily_id           PK
        int      meter_id           FK
        date     consumption_date
        numeric  volume_consumed_m3
        numeric  avg_flow_m3_h
        numeric  max_flow_m3_h
        int      sample_count
    }
    ems_w_weekly_consumption {
        serial   weekly_id          PK
        int      meter_id           FK
        int      year
        int      week_number
        numeric  volume_consumed_m3
    }
    ems_w_monthly_consumption {
        serial   monthly_id         PK
        int      meter_id           FK
        int      year
        int      month
        numeric  volume_consumed_m3
    }
    ems_w_annual_consumption {
        serial   annual_id          PK
        int      meter_id           FK
        int      year
        numeric  volume_consumed_m3
    }
    ems_w_system_parameters {
        serial   param_id        PK
        varchar  param_key
        varchar  param_value
        varchar  param_category
    }
```

**Źródło użytkowników** (User Source w bazie):

```mermaid
erDiagram
    scada_users ||--o{ scada_user_rl : "ma role"
    scada_roles ||--o{ scada_user_rl : "przypisana do"

    scada_users {
        serial   id        PK
        varchar  username
        varchar  passwd
        varchar  fname
        varchar  lname
    }
    scada_roles {
        serial   id        PK
        varchar  role_name
    }
    scada_user_rl {
        int  user_id  FK
        int  role_id  FK
    }
```

<details>
<summary><b>📋 Grupy tabel (rozwiń)</b></summary>

| Grupa | Tabele |
|-------|--------|
| **Konfiguracja** | `ems_w_locations`, `ems_w_meters`, `ems_w_system_parameters` |
| **Pomiary** | `ems_w_measurements` |
| **Agregacje** | `ems_w_daily_consumption`, `ems_w_weekly_consumption`, `ems_w_monthly_consumption`, `ems_w_annual_consumption` |
| **Użytkownicy** | `scada_users`, `scada_roles`, `scada_user_rl` |

</details>

---

## 🖥️ Ekrany aplikacji

| Ekran | Opis |
|-------|------|
| 🏠 **Dashboard** | Karty KPI dla M01 + podgląd na żywo wszystkich liczników, wykres trendu, status |
| 💧 **Liczniki** | Przegląd 18 liczników na żywo z symulatora + KPI wybranych liczników |
| 📊 **Raporty** | Zużycie dzienne / miesięczne / roczne + tabele z trybem „pełny ekran" |
| 🕓 **Historia** | Surowe pomiary z bazy (`ems_w_measurements`) + parametry archiwizacji |
| ⚙️ **Konfiguracja** | Tabele liczników i lokalizacji, podsumowania KPI |
| 📍 **Lokalizacje** | Zużycie dziś wg lokalizacji + szczegóły lokalizacji |
| 📈 **Analiza** | Filtry (lokalizacja + okres), trendy czasowe, wykresy porównawcze |

---

## 🔐 Użytkownicy i role

Role: **Administrator · Engineer · Operator · Analyst** · Strefy: **Plant Zone · Office Zone**

| Login | Imię i nazwisko | Rola | Hasło |
|-------|-----------------|------|-------|
| `admin1` | Adam Nowak | Administrator | = login |
| `engineer1` | Piotr Kowalski | Engineer | = login |
| `engineer2` | Anna Wiśniewska | Engineer | = login |
| `operator1` | Marcin Zieliński | Operator | = login |
| `operator2` | Tomasz Dąbrowski | Operator | = login |
| `analyst1` | Katarzyna Lewandowska | Analyst | = login |

> 🔑 Panel **Ignition Gateway**: `admin` / `admin`

---

## ⚙️ Backend

**Gateway Timer Scripts**

| Timer | Częstotliwość | Zadanie |
|-------|---------------|---------|
| `archiwizacja_pomiarow` | 10 s | Zapis pomiarów z tagów do bazy |
| `agregacja_dobowa` | 30 s | Agregacja dzienna (UPSERT) |
| `agregacja_tygodniowa_miesieczna_roczna` | 60 s | Agregacje tyg. / mies. / roczne |

**Named Queries** — wszystkie zapytania (33) zamknięte w Named Queries, pogrupowane wg funkcji:
`Aggregation` · `Dashboard` · `Analiza` · `Historia` · `Konfiguracja` · `Meter` · `Raporty`.

**Skrypt projektowy** — `ems_users` (odczyt użytkowników z User Source na ekran Konfiguracja).

---

## 👥 Autorzy

**Hubert Zdanowicz** · **Michał Tomys**
Grupa **31INF-PSI-SP** · *Ignition 8.3 z TTPSC — Digital Manufacturing*

<div align="center">
<br/>

*Zbudowane z 💧 w Ignition 8.3 + PostgreSQL*

</div>
