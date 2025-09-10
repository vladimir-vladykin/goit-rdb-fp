# goit-rdb-fp

## 1. Setup schema
```sql
DROP SCHEMA IF EXISTS pandemic;

CREATE SCHEMA pandemic;

USE pandemic;
```

## 2. Import and normalization
### Cases count
```sql
SELECT COUNT(*) as cases_count FROM infectious_cases;
```

### Normalization
```sql
DROP TABLE IF EXISTS cases_by_year;
DROP TABLE IF EXISTS regions;

CREATE TABLE regions (
  id INT NOT NULL AUTO_INCREMENT,
  region_name VARCHAR(50) NULL,
  region_code CHAR(10) NULL,
  PRIMARY KEY (id));

INSERT INTO regions (region_name, region_code)
SELECT Entity, Code 
FROM infectious_cases 
GROUP BY Entity, Code;



CREATE TABLE cases_by_year (
  id INT NOT NULL AUTO_INCREMENT,
  year YEAR NULL,
  region_id INT NULL,
  number_yaws INT NULL,
  number_polio INT NULL,
  number_cases_guinea_worm INT NULL,
  number_rabies INT NULL,
  number_malaria INT NULL,
  number_hiv INT NULL,
  number_tuberculosis INT NULL,
  number_smallpox INT NULL,
  number_cholera INT NULL,
  PRIMARY KEY (id),
  INDEX region_key_idx (region_id ASC) VISIBLE,
  CONSTRAINT region_key
    FOREIGN KEY (region_id)
    REFERENCES pandemic.regions (id)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION);


-- assume that some number are for hundreds of people, so multiple it to get absolute value
INSERT INTO cases_by_year
(year,
region_id,
number_yaws,
number_polio,
number_cases_guinea_worm,
number_rabies,
number_malaria,
number_hiv,
number_tuberculosis,
number_smallpox,
number_cholera)
SELECT 
	infectious_cases.Year as year,
    (SELECT id FROM regions WHERE regions.region_name = infectious_cases.Entity) as region_id,
    COALESCE(NULLIF(infectious_cases.Number_yaws, ''), 0) as number_yaws,
    COALESCE(NULLIF(infectious_cases.polio_cases, ''), 0) as polio_case,
    COALESCE(NULLIF(infectious_cases.cases_guinea_worm, ''), 0) as cases_guinea_worm,
    COALESCE(NULLIF(infectious_cases.Number_rabies, ''), 0) * 100 as number_rabies,
    COALESCE(NULLIF(infectious_cases.Number_malaria, ''), 0) as number_malaria,
    COALESCE(NULLIF(infectious_cases.Number_hiv, ''), 0) * 100 as number_hiv,
    COALESCE(NULLIF(infectious_cases.Number_tuberculosis, ''), 0) as number_tuberculosis,
    COALESCE(NULLIF(infectious_cases.Number_smallpox, ''), 0) as number_smallpox,
    COALESCE(NULLIF(infectious_cases.Number_cholera_cases, ''), 0) as number_cholera_cases
FROM pandemic.infectious_cases;
```


## 3. Analyze
```sql
SELECT 
	region_id, 
    AVG(number_rabies) as average_number_rabies,
    MIN(number_rabies) as minimum_number_rabies,
    MAX(number_rabies) as maximum_number_rabies
FROM cases_by_year 
GROUP BY region_id
ORDER BY average_number_rabies DESC
LIMIT 10
```

## 4. Years diff
```sql
SELECT 
	year as original_year,
    MAKEDATE(year, 1) as year_start,
    CURDATE() as today,
    TIMESTAMPDIFF(YEAR, MAKEDATE(year, 1), CURDATE()) as diff
FROM cases_by_year
```

## 5. Create own function
```sql
DROP FUNCTION IF EXISTS YEARSDIFF;


DELIMITER //

CREATE FUNCTION YEARSDIFF(year YEAR)
RETURNS INT
DETERMINISTIC 
NO SQL
BEGIN
	RETURN TIMESTAMPDIFF(YEAR, MAKEDATE(year, 1), CURDATE());
END //

DELIMITER ;


SELECT 
	year as original_year,
    YEARSDIFF(year) as diff
FROM cases_by_year
```


