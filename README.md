# carbon.sql
SQL for carbon emission calculation

## Scope GHG emission
```sql
WITH
a AS (
  SELECT
    TRIM("Scope") AS Scope,
    TRIM("emission_source") AS emission_source_key,
    "activity",
    CAST("amount" AS REAL) AS amount_raw,
    TRIM("unit") AS unit_raw,

    -- 酒精 L -> kg（只針對酒精）
    CASE
      WHEN (TRIM("unit") IN ('L','l','公升','liter','litre','Liter','Litre'))
       AND ("activity" LIKE '%酒精%')
      THEN CAST("amount" AS REAL) * 0.804
      ELSE CAST("amount" AS REAL)
    END AS amount_for_calc
  FROM "2603activity data"
  WHERE TRIM("Scope") IN ('Scope 1','Scope 2')
),
f AS (
  SELECT
    TRIM("emission source") AS emission_source_key,
    UPPER(TRIM("Gas")) AS gas,
    CAST("Factor" AS REAL) AS factor_value
  FROM "2603Factors"
),
g AS (
  SELECT
    UPPER(TRIM("GAS")) AS gas,
    CAST("GWP" AS REAL) AS gwp_value
  FROM "2603GWP"
)

-- Scope 1：CO2/CH4/N2O 走 *GWP
SELECT
  a.Scope,
  f.gas,
  ROUND(SUM(a.amount_for_calc * f.factor_value * g.gwp_value), 6) AS co2e_kg,
  ROUND(SUM(a.amount_for_calc * f.factor_value * g.gwp_value)/1000.0, 6) AS co2e_t
FROM a
JOIN f
  ON REPLACE(LOWER(a.emission_source_key),' ','_') = REPLACE(LOWER(f.emission_source_key),' ','_')
JOIN g
  ON f.gas = g.gas
WHERE a.Scope = 'Scope 1'
  AND f.gas IN ('CO2','CH4','N2O')
GROUP BY a.Scope, f.gas

UNION ALL

-- Scope 2：electricity 走 *factor（不乘 GWP）
SELECT
  a.Scope,
  'CO2e' AS gas,
  ROUND(SUM(a.amount_for_calc * f.factor_value), 6) AS co2e_kg,
  ROUND(SUM(a.amount_for_calc * f.factor_value)/1000.0, 6) AS co2e_t
FROM a
JOIN f
  ON REPLACE(LOWER(a.emission_source_key),' ','_') = REPLACE(LOWER(f.emission_source_key),' ','_')
WHERE a.Scope = 'Scope 2'
  AND REPLACE(LOWER(a.emission_source_key),' ','_') LIKE '%electric%'
GROUP BY a.Scope;
```

## Gas
```sql
WITH detail AS (
  -- 把上面 A 的 SELECT 改成只輸出 facility, gas, tCO2e 也行
  SELECT
    f.facility AS facility,
    f.gas AS gas,
    (a.amount_for_calc * f.factor_value * g.gwp)/1000.0 AS tCO2e
  FROM (
    SELECT
      REPLACE(LOWER(TRIM("emission_source")),' ','_') AS es_key,
      CASE
        WHEN (TRIM("unit") IN ('L','l','公升','liter','litre','Liter','Litre'))
         AND ("activity" LIKE '%酒精%')
        THEN CAST("amount" AS REAL) * 0.804
        ELSE CAST("amount" AS REAL)
      END AS amount_for_calc,
      UPPER(TRIM(gas)) AS gas_lock
    FROM "2603activity data"
  ) a
  JOIN (
    SELECT
      REPLACE(LOWER(TRIM("emission source")),' ','_') AS es_key,
      TRIM("活動/設施") AS facility,
      UPPER(TRIM("Gas")) AS gas,
      CAST("Factor" AS REAL) AS factor_value
    FROM "2603Factors"
  ) f
    ON a.es_key = f.es_key
   AND (
        (a.gas_lock IS NOT NULL AND a.gas_lock = f.gas)
        OR (a.gas_lock IS NULL AND f.gas IN ('CO2','CH4','N2O'))
      )
  JOIN (
    SELECT UPPER(TRIM("GAS")) AS gas, CAST("GWP" AS REAL) AS gwp
    FROM "2603GWP"
  ) g
    ON f.gas = g.gas
  WHERE f.gas IN ('CO2','CH4','N2O')
)
SELECT
  facility AS "活動/設施",
  ROUND(SUM(CASE WHEN gas='CO2' THEN tCO2e ELSE 0 END), 6) AS CO2_tCO2e,
  ROUND(SUM(CASE WHEN gas='CH4' THEN tCO2e ELSE 0 END), 6) AS CH4_tCO2e,
  ROUND(SUM(CASE WHEN gas='N2O' THEN tCO2e ELSE 0 END), 6) AS N2O_tCO2e,
  ROUND(SUM(tCO2e), 6) AS total_tCO2e
FROM detail
GROUP BY facility
ORDER BY total_tCO2e DESC;
```

##percentage
```sql
WITH calc AS (
  SELECT
    f."活動/設施" AS facility,
    (a.amount_for_calc * f.factor_value * g.gwp)/1000.0 AS tCO2e
  FROM (
    SELECT
      REPLACE(LOWER(TRIM("emission_source")),' ','_') AS es_key,
      UPPER(TRIM(gas)) AS gas_lock,
      CASE
        WHEN (TRIM("unit") IN ('L','l','公升','liter','litre','Liter','Litre'))
         AND ("activity" LIKE '%酒精%')
        THEN CAST("amount" AS REAL) * 0.804
        ELSE CAST("amount" AS REAL)
      END AS amount_for_calc
    FROM "2603activity data"
  ) a
  JOIN (
    SELECT
      REPLACE(LOWER(TRIM("emission source")),' ','_') AS es_key,
      TRIM("活動/設施") AS "活動/設施",
      UPPER(TRIM("Gas")) AS gas,
      CAST("Factor" AS REAL) AS factor_value
    FROM "2603Factors"
  ) f
    ON a.es_key = f.es_key
   AND (
        (a.gas_lock IS NOT NULL AND a.gas_lock = f.gas)
        OR
        (a.gas_lock IS NULL AND f.gas IN ('CO2','CH4','N2O'))
       )
  JOIN (
    SELECT
      UPPER(TRIM("GAS")) AS gas,
      CAST("GWP" AS REAL) AS gwp
    FROM "2603GWP"
  ) g
    ON f.gas = g.gas
  WHERE f.gas IN ('CO2','CH4','N2O')
)

SELECT
  facility AS "活動/設施",
  ROUND(SUM(tCO2e), 2) AS total_tCO2e,
  ROUND(SUM(tCO2e) * 100.0 / (SELECT SUM(tCO2e) FROM calc), 2) AS percentage
FROM calc
GROUP BY facility
ORDER BY total_tCO2e DESC;
```
