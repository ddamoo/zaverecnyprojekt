# ELT proces datasetu Flight Status

Tento repozitár predstavuje implementáciu **ELT procesu v Snowflake** a vytvorenie dátového skladu so schémou Star Schema pre dáta o leteckých letoch a ich statusoch. Projekt sa zameriava na analýzu počtu letov, statusov letov a porovnanie aktivít leteckých dopravcov. Výsledný dátový model umožňuje multidimenzionálnu analýzu a vizualizáciu kľúčových metrík.

Cieľom projektu je demonštrovať správnu implementáciu ELT procesu, tvorbu dimenzií a faktovej tabuľky a následnú analýzu dát.

---

## 1. Úvod a popis zdrojových dát

Analýza sa zameriava na **dáta o letoch, dopravcoch a stave letov**. Cieľom je porozumieť:

- počtu letov jednotlivých dopravcov,
- distribúcii statusov letov,
- aktivite dopravcov v jednotlivé dni,
- trendom meškaní alebo zrušení letov.

Zdrojové dáta pochádzajú zo Snowflake Marketplace (OAG Flight Status Data Sample) a obsahujú nasledujúce tabuľky:

- `FLIGHT_STATUS_LATEST_SAMPLE` – surové dáta o letoch vrátane IATA a ICAO kódov, čísla letu, typu letu, statusu, plánovaného času odletu.

Účelom ELT procesu bolo tieto dáta pripraviť, transformovať a sprístupniť pre viacdimenzionálnu analýzu.

---

### 1.1 Dátová architektúra

Surové dáta sú usporiadané v relačnom modeli, znázornenom na entitno-relačnom diagrame (ERD).  

**ERD Schema**  
<img width="1916" height="953" alt="ERD" src="https://github.com/user-attachments/assets/2b1eba6e-a075-49c2-8667-5c889535d43b" />


---

## 2. Dimenzionálny model

Pre analýzu bola navrhnutá **schéma hviezdy (Star Schema)** podľa Kimballovej metodológie, ktorá obsahuje jednu faktovú tabuľku `fact_flight_status` a štyri dimenzie:

- **dim_carrier** – obsahuje IATA a ICAO kódy dopravcov.  
- **dim_flight** – obsahuje čísla letov a typy letov.  
- **dim_status** – obsahuje rôzne statusy letu (napr. on time, delayed, cancelled).  
- **dim_date** – informácie o dátumoch letov (deň, mesiac, rok).

**Star Schema**  
<img width="982" height="607" alt="starschema" src="https://github.com/user-attachments/assets/13043f20-fe80-4b30-a6a6-875474504da4" />

---

## 3. ELT proces v Snowflake

ETL proces pozostáva z troch hlavných fáz: **Extract**, **Load** a **Transform**. Tento proces bol implementovaný v Snowflake a pripravil zdrojové dáta zo staging vrstvy do viacdimenzionálneho modelu vhodného na analýzu a vizualizáciu.

### 3.1 Extract (Extrahovanie dát)

Surové dáta zo Snowflake Marketplace boli uložené do staging tabuliek:

~~~sql
CREATE OR REPLACE TABLE stg_flight_status AS
SELECT *
FROM OAG_FLIGHT_STATUS_DATA_SAMPLE.PUBLIC.FLIGHT_STATUS_LATEST_SAMPLE;
~~~




3.2 Load (Načítanie dát)

Dáta zo staging tabuliek boli načítané do dimenzií:

CREATE OR REPLACE TABLE dim_carrier AS
SELECT DISTINCT
    IATA_CARRIER_CODE,
    ICAO_CARRIER_CODE,
    CURRENT_DATE AS valid_from,
    NULL AS valid_to
FROM stg_flight_status;

CREATE OR REPLACE TABLE dim_flight AS
SELECT DISTINCT
    FLIGHT_NUMBER,
    FLIGHT_TYPE
FROM stg_flight_status;

CREATE OR REPLACE TABLE dim_status AS
SELECT DISTINCT
    STATUS_KEY
FROM stg_flight_status;

CREATE OR REPLACE TABLE dim_date AS
SELECT DISTINCT
    CAST(SCHEDULED_DEPARTURE_DATE_LOCAL AS DATE) AS FLIGHT_DATE,
    YEAR(CAST(SCHEDULED_DEPARTURE_DATE_LOCAL AS DATE)) AS year,
    MONTH(CAST(SCHEDULED_DEPARTURE_DATE_LOCAL AS DATE)) AS month,
    DAY(CAST(SCHEDULED_DEPARTURE_DATE_LOCAL AS DATE)) AS day
FROM stg_flight_status;




3.3 Transform (Transformácia dát)

Faktová tabuľka bola vytvorená pomocou window functions a prepojenia na dimenzie:

CREATE OR REPLACE TABLE fact_flight_status AS
WITH carrier_counts AS (
    SELECT
        cs.STATUS_KEY,
        cs.IATA_CARRIER_CODE,
        cs.ICAO_CARRIER_CODE,
        cs.FLIGHT_NUMBER,
        cs.FLIGHT_TYPE,
        d.FLIGHT_DATE,
        COUNT(*) OVER (PARTITION BY cs.IATA_CARRIER_CODE) AS carrier_flight_count
    FROM stg_flight_status cs
    JOIN dim_carrier c ON cs.IATA_CARRIER_CODE = c.IATA_CARRIER_CODE
    JOIN dim_flight f ON cs.FLIGHT_NUMBER = f.FLIGHT_NUMBER
    JOIN dim_status s ON cs.STATUS_KEY = s.STATUS_KEY
    JOIN dim_date d ON CAST(cs.SCHEDULED_DEPARTURE_DATE_LOCAL AS DATE) = d.FLIGHT_DATE
)
SELECT
    ROW_NUMBER() OVER (ORDER BY FLIGHT_DATE) AS flight_status_sk,
    FLIGHT_DATE,
    IATA_CARRIER_CODE,
    FLIGHT_NUMBER,
    STATUS_KEY,
    carrier_flight_count,
    RANK() OVER (
        PARTITION BY FLIGHT_DATE
        ORDER BY carrier_flight_count DESC
    ) AS daily_carrier_rank
FROM carrier_counts;


4. Vizualizácia dát

Top dopravcov podľa počtu letov
<img width="1629" height="354" alt="prvygraf" src="https://github.com/user-attachments/assets/c3d27dca-1815-472a-8cca-daeac5b59952" />
Ukazuje, ktorí dopravcovia majú najviac letov v datasetu. Pomáha identifikovať hlavné letecké spoločnosti a ich podiel na prevádzke.

Počet letov podľa statusu
<img width="1630" height="347" alt="druhygraf" src="https://github.com/user-attachments/assets/b8d71e7f-da00-4309-a672-3997bf9156af" />
Ukazuje trend počtu letov v čase. Pomáha sledovať sezónnosť, vrcholy prevádzky alebo poklesy v leteckej doprave.

Denná aktivita dopravcov
<img width="1626" height="344" alt="tretigraf" src="https://github.com/user-attachments/assets/d80b51a2-9446-412d-8dd8-cad4e21b4dcf" />
Zobrazuje podiel jednotlivých statusov letov (napr. On Time, Delayed, Cancelled). Pomáha pochopiť kvalitu prevádzky a identifikovať najčastejšie problémy.

Rank dopravcov podľa počtu letov na deň
<img width="1617" height="347" alt="stvrtygraf" src="https://github.com/user-attachments/assets/3e6d341b-67cd-49a5-a944-41c68d259c17" />
Ukazuje kumulatívny počet letov pre každého dopravcu v čase. Pomáha sledovať rast prevádzky a porovnávať dopravcov medzi sebou.

Distribúcia statusov letov pre konkrétneho dopravcu
<img width="1629" height="333" alt="piatygraf" src="https://github.com/user-attachments/assets/74ab154a-cdff-44f1-a3ed-d22142d26d79" />
Ukazuje, ktorí dopravcovia mali v jednotlivých dňoch najviac letov. Pomáha sledovať denné poradie a výkonnosť dopravcov.


Autor: Damian Bartek


