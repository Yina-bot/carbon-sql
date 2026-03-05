# carbon.sql
SQL for carbon emission calculation

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
