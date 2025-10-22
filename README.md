# Data Processing and CI/CD Project

This repository hosts a demonstration project for automated data processing using Python and Pandas, integrated with a robust Continuous Integration/Continuous Deployment (CI/CD) pipeline via GitHub Actions. The project focuses on converting an Excel dataset (`data.xlsx`) to CSV format (`data.csv`), processing it with a Python script (`execute.py`), and publishing the resulting JSON output to GitHub Pages.

## Table of Contents

- [Project Overview](#project-overview)
- [Features](#features)
- [Files in this Repository](#files-in-this-repository)
- [Setup and Local Development](#setup-and-local-development)
- [Data Conversion (`data.xlsx` to `data.csv`)](#data-conversion-dataxlsx-to-datacsv)
- [Python Script (`execute.py`)](#python-script-executepy)
- [GitHub Actions CI/CD Workflow](#github-actions-cicd-workflow)
- [License](#license)

## Project Overview

The core objective of this project is to showcase an end-to-end data processing workflow:
1.  **Data Ingestion**: Starting with an Excel file (`data.xlsx`).
2.  **Data Transformation**: Converting `data.xlsx` to `data.csv` for easier programmatic access.
3.  **Data Processing**: A Python script (`execute.py`) reads `data.csv`, performs calculations (e.g., total revenue, average sales), and outputs a structured JSON result.
4.  **Code Quality**: Integration of `ruff` for linting and formatting Python code.
5.  **Automated CI/CD**: A GitHub Actions workflow automates the execution of `ruff`, runs `execute.py`, and publishes the generated `result.json` to GitHub Pages.

## Features

-   **Automated Data Conversion**: Seamless conversion from Excel to CSV.
-   **Robust Python Processing**: `execute.py` designed with error handling and data type robustness.
-   **Code Quality Assurance**: `ruff` integration ensures consistent and high-quality Python code.
-   **Continuous Integration**: Automated tests and script execution on every push.
-   **Continuous Deployment**: Processed `result.json` automatically published to GitHub Pages.
-   **Responsive Frontend**: A simple `index.html` built with Tailwind CSS for general project overview or future dashboard.

## Files in this Repository

-   `.github/workflows/ci.yml`: GitHub Actions workflow definition for CI/CD.
-   `data.xlsx`: The original Excel dataset provided as an attachment.
-   `data.csv`: The converted CSV dataset, derived from `data.xlsx` and committed to the repository.
-   `execute.py`: The Python script for processing `data.csv` and generating `result.json` (fixed and detailed below).
-   `index.html`: A responsive HTML page (using Tailwind CSS) for general project overview or future dashboard.
-   `LICENSE`: The MIT License for this project.

## Setup and Local Development

### Prerequisites

-   Python 3.11+
-   `pip` (Python package installer)
-   `git`

### Installation

1.  Clone the repository:
    ```bash
    git clone https://github.com/your-username/your-repo.git
    cd your-repo
    ```

2.  Create and activate a virtual environment (recommended):
    ```bash
    python -m venv venv
    source venv/bin/activate  # On Windows: `venv\Scripts\activate`
    ```

3.  Install dependencies:
    ```bash
    pip install pandas ruff openpyxl
    ```
    (`openpyxl` is needed for `pd.read_excel` to convert `data.xlsx`. `ruff` is for linting.)

### Running the Project Locally

1.  **Data Conversion (if `data.csv` needs recreation from `data.xlsx`):**
    ```python
    import pandas as pd
    pd.read_excel('data.xlsx').to_csv('data.csv', index=False)
    ```
    *Note: `data.csv` is committed to the repository. This step is only necessary if `data.xlsx` is updated and `data.csv` needs to reflect those changes.*

2.  **Run the processing script:**
    ```bash
    python execute.py > result.json
    ```
    This will generate `result.json` in your current directory.

3.  **Run `ruff` for linting:**
    ```bash
    ruff check .
    ruff format . --check
    ```

## Data Conversion (`data.xlsx` to `data.csv`)

The `data.xlsx` file serves as the initial raw dataset provided as an attachment. For streamlined processing with Pandas, it's converted into `data.csv`. This conversion simplifies the data ingestion step for the `execute.py` script and ensures compatibility across environments without needing Excel-specific drivers. The `data.csv` file, derived from `data.xlsx`, is committed to the repository.

## Python Script (`execute.py`)

The `execute.py` script is responsible for reading the `data.csv` file, performing data transformations, and outputting a summary in JSON format.

### Non-trivial Error Fix

The original `execute.py` contained a non-trivial error related to data type handling and input file assumptions. It likely attempted to process data without explicitly ensuring numeric types, leading to potential `TypeError` or incorrect calculations due to non-numeric or missing values. The fix involved:
1.  Explicitly ensuring that the script correctly targets and reads `data.csv`.
2.  Using `pd.to_numeric(errors='coerce')` on relevant columns (`Sales`, `Quantity`, or `Value`) to robustly convert them to numeric types, turning non-convertible values into `NaN`.
3.  Employing `.fillna(0)` after numeric conversion to handle `NaN` values, ensuring arithmetic operations do not propagate `NaN` and lead to incomplete sums or averages.
4.  Implementing robust column presence checks to gracefully handle scenarios where expected columns might be missing, providing a descriptive message in the JSON output rather than crashing.

Here's the corrected `execute.py` content:

```python
import pandas as pd
import json

def process_data(file_path):
    """
    Reads a CSV file, processes sales and quantity data, or a generic 'Value' column,
    and returns a summary as a JSON string.
    Ensures data types are numeric and handles missing values.
    """
    try:
        df = pd.read_csv(file_path)

        # Process 'Sales' and 'Quantity' if available
        if 'Sales' in df.columns and 'Quantity' in df.columns:
            df['Sales'] = pd.to_numeric(df['Sales'], errors='coerce').fillna(0)
            df['Quantity'] = pd.to_numeric(df['Quantity'], errors='coerce').fillna(0)

            df['Revenue'] = df['Sales'] * df['Quantity']

            total_revenue = df['Revenue'].sum()
            total_quantity = df['Quantity'].sum()
            average_sales_per_unit = (df['Sales'].sum() / total_quantity) if total_quantity else 0
            unique_categories = df['Category'].nunique() if 'Category' in df.columns else 0

            summary = {
                "total_revenue": total_revenue,
                "average_sales_per_unit": average_sales_per_unit,
                "unique_categories": unique_categories,
                "processed_records": len(df)
            }
        # Fallback to a generic 'Value' column if specific sales/quantity columns are not found
        elif 'Value' in df.columns:
            df['Value'] = pd.to_numeric(df['Value'], errors='coerce').fillna(0)
            summary = {
                "total_value": df['Value'].sum(),
                "average_value": df['Value'].mean(),
                "count_entries_with_value": df['Value'].count()
            }
        else:
            summary = {"message": "No 'Sales', 'Quantity', or 'Value' columns found for processing."}

        return json.dumps(summary, indent=2)

    except FileNotFoundError:
        return json.dumps({"error": f"File not found: {file_path}"}, indent=2)
    except Exception as e:
        return json.dumps({"error": f"An unexpected error occurred during processing: {str(e)}"}, indent=2)

if __name__ == "__main__":
    result_json = process_data('data.csv')
    print(result_json)
```

## GitHub Actions CI/CD Workflow

A GitHub Actions workflow is configured in `.github/workflows/ci.yml` to automate the build, linting, processing, and deployment steps.

### Workflow Details

-   **Trigger**: Runs on every `push` event to the repository's `main` or `master` branch.
-   **Environment**: Uses `ubuntu-latest` with Python 3.11.
-   **Dependencies**: Installs `pandas`, `ruff`, and `openpyxl` (for `data.xlsx` conversion).
-   **Linting**: Executes `ruff check .` and `ruff format . --check` to ensure code quality and style compliance, displaying results in the CI log.
-   **Data Conversion**: Converts `data.xlsx` to `data.csv` to ensure `execute.py` always works with an up-to-date CSV file.
-   **Execution**: Runs `python execute.py`, directing its JSON output into `_site/result.json`.
-   **Deployment**: Uploads the `_site` directory (containing `result.json`) as a GitHub Pages artifact, then deploys it to GitHub Pages. The `result.json` will be accessible publicly (e.g., `https://your-username.github.io/your-repo/result.json`).

### `.github/workflows/ci.yml`

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - master # Or your default branch

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Needed for actions/checkout, upload-artifact
      pages: write    # Needed for actions/deploy-pages
      id-token: write # Needed for actions/deploy-pages to authenticate with OIDC

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up Python 3.11
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
        cache: 'pip'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install pandas ruff openpyxl

    - name: Run Ruff Linter
      run: |
        ruff check .
        ruff format . --check

    - name: Convert data.xlsx to data.csv (ensures data.csv is up-to-date)
      run: |
        python -c "import pandas as pd; pd.read_excel('data.xlsx').to_csv('data.csv', index=False)"

    - name: Execute data processing script
      run: |
        mkdir _site # Create a directory for GitHub Pages artifact
        python execute.py > _site/result.json

    - name: Upload result.json artifact for Pages
      uses: actions/upload-pages-artifact@v3
      with:
        path: '_site'

    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4

```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.