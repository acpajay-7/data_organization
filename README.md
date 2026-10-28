Enterprise B2B & MCA Director Data Pipeline: Cleaning, Normalization & StandardizationA production-grade Python ETL pipeline built for ingesting, standardizing, imputing, and cleaning high-dimensional B2B enterprise datasets, IT industry leadership lists, and MCA (Ministry of Corporate Affairs) registered director databases.1. Project OverviewRaw corporate and director datasets sourced from directories and statutory registries often exhibit severe structural and semantic defects:High Nullity Columns: Sparse tracking metadata (Email Confidence, Tertiary Email, Other Phone, Qualify Contact) containing 100% missing values.Inconsistent Header Conventions: Non-standard casing, punctuation, and whitespaces (e.g., 'Company Name for Emails', 'Director Email Address ').Uncleaned Phone Numbers & Missing Values: Free-form text phones with escape quotes (e.g., '+91 80025 36500), missing corporate lines, and unindexed contact channels.Mixed Casing & Whitespace Padding: Mixed casing in director names, company names, cities, and email IDs affecting deduplication and query joins.Missing Numeric Metrics: Unfilled financial capital fields (Authorized Capital, Paid Capital) in statutory MCA filings.This pipeline automates column pruning, snake_case schema normalization, domain-specific missing value imputation, string stripping/lowercasing, and standardized .xlsx exports.2. Processed Datasets & SchemasSource DatasetDomain / EntitiesKey TransformationsExport Targetindian_businessman_2026.xlsxBusiness Owners & FoundersDropped 100% null columns (7 cols), removed redundant mobile_phone, imputed contact defaults, normalized strings to lowercase.cleaned_data4.xlsxit_information_technology_39185.xlsxIT Executives & C-Suite (39k+ rows)Renamed columns to canonical snake_case, cast mobile to string, stripped whitespace, lowercased categorical fields.cleaned_data5.xlsxmca_companies_directors_mobile_database.xlsxMCA Registered Directors (1.5k rows)Filled missing authorized_capital & paid_capital with 'unknown', lowercased CIN, DIN, names, addresses, and email IDs.cleaned_data6.xlsx3. Data Cleaning Pipeline ArchitectureRaw Excel Files (.xlsx)
        │
        ▼
[ 1. Null-Column Pruning ]
        │  ──> Drops columns where all values are NaN (isna().all())
        ▼
[ 2. Header Standardization ]
        │  ──> Trim spaces, convert to lowercase, replace ' ' with '_', regex clean non-alphanumeric
        ▼
[ 3. Domain Imputation ]
        │  ──> Phone missing values -> '0000000000' / 'unknown'
        │  ──> Geography missing values -> 'unknown'
        │  ──> Capital metrics -> 'unknown' / fallback defaults
        ▼
[ 4. Text & Type Normalization ]
        │  ──> Convert text fields to lowercase, strip trailing/leading spaces
        │  ──> Safe string extraction for identifiers (CIN, DIN, Mobile)
        ▼
Cleaned Enterprise Datasets (.xlsx)
4. Key Implementation DetailsA. Dynamic Pruning of Completely Empty ColumnsRemoves empty tracking tags without hardcoding column names:# Drop columns where all entries are NaN
df = df.dropna(how='all', axis=1)
B. Universal Column NormalizationConverts arbitrary multi-word headers into uniform programmatic keys:df.columns = (
    df.columns
    .str.strip()
    .str.lower()
    .str.replace(' ', '_')
    .str.replace(r'[^\w]', '', regex=True)
)
C. Contextual Missing-Value ImputationApplies domain-appropriate fallbacks instead of indiscriminately dropping incomplete contact entries:imputation_rules = {
    'corporate_phone': 'unknown',
    'company_phone': '0000000000',
    'state': 'unknown',
    'company_city': 'unknown',
    'company_state': 'unknown',
    'company_country': 'unknown',
    'secondary_email_status': 'na',
    'secondary_email_verification_source': 'na',
    'authorized_capital': 'unknown',
    'paid_capital': 'unknown'
}

for col, fill_value in imputation_rules.items():
    if col in df.columns:
        df[col] = df[col].fillna(fill_value)
D. Universal Text NormalizationGuarantees clean joins across downstream CRM systems, databases, and search queries:string_cols = df.select_dtypes(include=['object']).columns

for col in string_cols:
    df[col] = df[col].astype(str).str.strip().str.lower()
5. Repository Structure├── data/
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
6. Setup and InstallationPrerequisitesPython 3.10+Google Colab or Local Jupyter EnvironmentInstall Dependenciespip install pandas openpyxl xlrd
Run PipelineOpen notebooks/data_cleaning_and_imputation.ipynb in Google Colab or VS Code Jupyter Extension and execute all cells sequentially.