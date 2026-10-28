# Enterprise B2B & MCA Director Data Pipeline
### Data Cleaning, Normalization & Standardization

A Python ETL pipeline for ingesting, standardizing, imputing, and cleaning B2B enterprise datasets, IT industry leadership lists, and director records from India's Ministry of Corporate Affairs (MCA).

The pipeline uses **Pandas** and Excel-processing libraries to improve data consistency and prepare `.xlsx` files for analysis, deduplication, search, and downstream CRM workflows.

## 1. Project Overview

Raw business directories and statutory datasets can contain structural and semantic issues that make analysis difficult, including:

- **Completely empty columns:** Tracking fields such as email confidence, tertiary email, other phone, and qualification metadata may contain no usable values.
- **Inconsistent headers:** Column names may contain extra spaces, mixed casing, punctuation, or inconsistent naming conventions.
- **Unstandardized contact details:** Phone numbers may contain leading quotes, inconsistent formatting, or missing values.
- **Inconsistent text formatting:** Names, company details, cities, and email addresses may have mixed casing or leading/trailing spaces.
- **Missing financial fields:** MCA records may have missing authorized or paid capital values.
- **Inconsistent identifiers:** CIN, DIN, and mobile-number fields may need careful type handling to preserve their values.

This project automates column pruning, header normalization, configurable missing-value imputation, text normalization, and Excel exports.

> **Data quality note:** Placeholder values such as `unknown`, `na`, or `0000000000` indicate missing data; they are not verified contact details.

## 2. Datasets and Outputs

| Source dataset | Domain / entities | Main transformations | Output file |
|---|---|---|---|
| `indian_businessman_2026.xlsx` | Business owners and founders | Removed completely empty columns; removed the redundant `mobile_phone` field; filled selected missing values; normalized text | `cleaned_data4.xlsx` |
| `it_information_technology_39185.xlsx` | IT executives and C-suite contacts; approximately 39,185 records, as indicated by the filename | Standardized headers; converted `mobile` to string where applicable; stripped whitespace; lowercased text fields | `cleaned_data5.xlsx` |
| `mca_companies_directors_mobile_database.xlsx` | MCA-registered companies and directors; approximately 1,500 records, as described for this dataset | Filled selected missing capital values; normalized CIN, DIN, names, addresses, and email fields | `cleaned_data6.xlsx` |

*Record counts and transformation summaries should be confirmed against the actual source files and notebook output.*

## 3. Pipeline Architecture

```text
Raw Excel files (.xlsx)
        |
        v
1. Remove completely empty columns
        |
        v
2. Standardize column headers
        |
        v
3. Apply configured missing-value rules
        |
        v
4. Normalize text and selected data types
        |
        v
5. Export cleaned Excel files (.xlsx)
        |
        v
Cleaned datasets for analysis and downstream workflows
```

## 4. Cleaning Steps and Implementation

### A. Remove Completely Empty Columns

Drops columns in which every value is missing, without requiring column names to be hardcoded.

```python
df = df.dropna(axis=1, how="all")
```

This removes fully empty columns while retaining columns that contain at least one non-missing value.

### B. Standardize Column Names

Trims whitespace, converts headers to lowercase, replaces spaces with underscores, and removes punctuation that is not part of a word character.

```python
df.columns = (
    df.columns
    .str.strip()
    .str.lower()
    .str.replace(r"\s+", "_", regex=True)
    .str.replace(r"[^\w]", "", regex=True)
)
```

For example, `Company Name for Emails` becomes `company_name_for_emails`.

**Note:** This transformation can cause different original headers to normalize to the same name. Check for duplicate column names after normalization.

### C. Apply Missing-Value Rules

Uses configurable fallback values for selected columns instead of automatically dropping every row with an incomplete contact record.

```python
imputation_rules = {
    "corporate_phone": "unknown",
    "company_phone": "0000000000",
    "state": "unknown",
    "company_city": "unknown",
    "company_state": "unknown",
    "company_country": "unknown",
    "secondary_email_status": "na",
    "secondary_email_verification_source": "na",
    "authorized_capital": "unknown",
    "paid_capital": "unknown",
}

for col, fill_value in imputation_rules.items():
    if col in df.columns:
        df[col] = df[col].fillna(fill_value)
```

Only columns present in the DataFrame are modified.

**Important:** A value such as `0000000000` may be mistaken for a real phone number, so a text placeholder is often safer unless a downstream system requires a numeric-looking format.

### D. Normalize Text Fields

Strips leading and trailing whitespace and converts text values to lowercase.

```python
string_cols = df.select_dtypes(include=["object", "string"]).columns

for col in string_cols:
    df[col] = df[col].apply(
        lambda value: value.strip().lower()
        if isinstance(value, str)
        else value
    )
```

This can improve matching and joining across datasets. Apply it selectively if capitalization is meaningful for display or if identifiers must preserve their original representation.

### E. Preserve Contact Numbers and Identifiers

Mobile numbers, CINs, and DINs should generally be treated as strings when leading zeros, formatting, or exact identifier representation matters.

```python
identifier_cols = ["mobile", "mobile_phone", "cin", "din"]

for col in identifier_cols:
    if col in df.columns:
        df[col] = df[col].astype("string").str.strip()
```

Converting values that were already imported as floating-point numbers to strings cannot always recover lost leading zeros or the original formatting. For best results, import these columns as strings from the source Excel file when possible.

### F. Export Cleaned Data

```python
from pathlib import Path

output_dir = Path("data/cleaned")
output_dir.mkdir(parents=True, exist_ok=True)

df.to_excel(output_dir / "cleaned_output.xlsx", index=False)
```

Replace `cleaned_output.xlsx` with the required output filename for each dataset.

## 5. Repository Structure

```text
.
├── data/
│   ├── raw/
│   │   ├── indian_businessman_2026.xlsx
│   │   ├── it_information_technology_39185.xlsx
│   │   └── mca_companies_directors_mobile_database.xlsx
│   └── cleaned/
│       ├── cleaned_data4.xlsx
│       ├── cleaned_data5.xlsx
│       └── cleaned_data6.xlsx
├── notebooks/
│   └── data_cleaning_and_imputation.ipynb
├── requirements.txt
└── README.md
```

The structure above is a suggested layout; adjust it to match the actual files and directories in your repository.

## 6. Requirements

- Python 3.10 or newer
- Pandas
- OpenPyXL
- Jupyter Notebook or Google Colab

Install dependencies:

```bash
pip install pandas openpyxl jupyter
```

If your source files require legacy Excel `.xls` support, you may also need:

```bash
pip install xlrd
```

A minimal `requirements.txt` could contain:

```text
pandas
openpyxl
jupyter
```

Add `xlrd` only if it is required by your input files.

## 7. How to Run

1. Clone or download the repository.
2. Place the original Excel datasets in `data/raw/`.
3. Install the dependencies listed above.
4. Open `notebooks/data_cleaning_and_imputation.ipynb` in Jupyter Notebook, VS Code, or Google Colab.
5. Update input and output paths if your directory layout differs.
6. Run the notebook cells in order.
7. Check the generated files in `data/cleaned/`.
8. Validate row counts, column names, missing values, duplicates, and data types before using the outputs.

Example for a local Jupyter environment:

```bash
jupyter notebook
```

## 8. Validation Checklist

Before treating the cleaned files as production-ready, verify:

- [ ] Completely empty columns have been removed.
- [ ] Column names are standardized and unique.
- [ ] Missing-value placeholders are used consistently.
- [ ] Mobile numbers and statutory identifiers have been checked for unintended changes.
- [ ] Duplicate records have been assessed using suitable business keys.
- [ ] Row counts before and after cleaning have been compared.
- [ ] Excel outputs open correctly and contain the expected columns.
- [ ] Sample records have been checked against their source data.

## 9. Data Quality and Responsible Use

- Missing-value imputation does not verify whether a contact detail is correct or current.
- Do not treat placeholder phone numbers or email values as real contact information.
- Preserve source data and write cleaned outputs to separate files.
- Where possible, retain source references and record the transformations applied.
- Use business and director contact information responsibly and in accordance with applicable privacy, data-protection, and outreach requirements.

## 10. Future Improvements

Potential enhancements include:

- Automated duplicate detection and configurable deduplication keys.
- Phone-number and email-format validation.
- Data-quality reports before and after cleaning.
- Logging and exception handling for failed input files.
- Configurable rules per dataset rather than one shared rule set.
- Automated batch processing of all Excel files in `data/raw/`.
- Unit tests for header normalization, imputation, and export behavior.

## 11. License and Data Attribution

Add a license file if you intend to distribute this project. Confirm that you have the right to process and share each source dataset, and document its origin and any applicable usage restrictions.
