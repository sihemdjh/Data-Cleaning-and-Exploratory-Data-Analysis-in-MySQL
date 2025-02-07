# Data Cleaning and Exploratory Data Analysis (EDA) in MySQL

This repository contains scripts and queries for cleaning and analyzing layoff data using MySQL. Below are the steps for data cleaning and Exploratory Data Analysis (EDA), along with corresponding SQL queries.

## 1. Data Cleaning

### 1.1 Remove Duplicates

Duplicates can distort the data. Use the `ROW_NUMBER()` function to identify and remove duplicate rows based on key columns.

```sql
WITH duplicate_cte AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`
           ) AS row_num
    FROM layoffs_staging
)
DELETE FROM layoffs_staging
WHERE id IN (
    SELECT id FROM duplicate_cte WHERE row_num > 1
);

```
### 1.2 Standardize the Data

#### Trim extra spaces from text fields:
```sql
UPDATE layoffs_staging2
SET company = TRIM(company);
```

#### Normalize inconsistent values:
```sql
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';
```

#### Convert date formats:
```sql
UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');

ALTER TABLE layoffs_staging2 MODIFY COLUMN `date` DATE;
```

### 1.3 Handle Null or Blank Values

#### Replace null or blank values with appropriate substitutes:
```sql
UPDATE layoffs_staging2
SET industry = NULL
WHERE industry = '';
```

#### Fill missing values using data from other rows:
```sql
UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2 ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE t1.industry IS NULL AND t2.industry IS NOT NULL;
```

### 1.4 Remove Unnecessary Columns or Rows

#### Drop irrelevant columns:
```sql
ALTER TABLE layoffs_staging2 DROP COLUMN row_num;
```

#### Remove rows with missing critical data:
```sql
DELETE FROM layoffs_staging2
WHERE total_laid_off IS NULL AND percentage_laid_off IS NULL;
```

## 2. Exploratory Data Analysis (EDA)

### 2.1 Summary Statistics

#### Maximum layoffs and percentage laid off:
```sql
SELECT MAX(total_laid_off), MAX(percentage_laid_off) FROM layoffs_staging2;
```

#### Company with the most layoffs:
```sql
SELECT company, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY company
ORDER BY 2 DESC;
```

### 2.2 Trends and Patterns

#### Monthly layoffs trend:
```sql
SELECT SUBSTRING(`date`, 1, 7) AS `month`, SUM(total_laid_off)
FROM layoffs_staging2
WHERE SUBSTRING(`date`, 1, 7) IS NOT NULL
GROUP BY `month`
ORDER BY 1 ASC;
```

#### Rolling total of layoffs:
```sql
WITH rolling_total AS (
    SELECT SUBSTRING(`date`, 1, 7) AS `month`, SUM(total_laid_off) AS total_off
    FROM layoffs_staging2
    WHERE SUBSTRING(`date`, 1, 7) IS NOT NULL
    GROUP BY `month`
    ORDER BY 1 ASC
)
SELECT `month`, total_off, SUM(total_off) OVER (ORDER BY `month`) AS rolling_total
FROM rolling_total;
```

### 2.3 Grouped Analysis

#### Layoffs by country:
```sql
SELECT country, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY country
ORDER BY 2 DESC;
```

#### Top companies by layoffs per year:
```sql
WITH company_year AS (
    SELECT company, YEAR(`date`) AS years, SUM(total_laid_off) AS total_laid_off
    FROM layoffs_staging2
    GROUP BY company, years
), company_year_rank AS (
    SELECT *, DENSE_RANK() OVER (PARTITION BY years ORDER BY total_laid_off DESC) AS ranking
    FROM company_year
    WHERE years IS NOT NULL
)
SELECT *
FROM company_year_rank
WHERE ranking <= 5;
```
