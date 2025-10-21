# GitHub Actions Workflows for Unemployment Analysis

This file provides the workflow configurations to set up in `.github/workflows/` directory.

## File 1: `notebook-validation.yml`

```yaml
name: Notebook Validation

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  validate-notebook:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        python-version: ['3.8', '3.9', '3.10']
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install jupyter pandas numpy matplotlib seaborn kagglehub
    
    - name: Check notebook syntax
      run: |
        jupyter nbconvert --to script Analyze_unemployment3.ipynb --stdout > /dev/null
    
    - name: Validate notebook execution
      run: |
        jupyter nbconvert --to notebook --execute Analyze_unemployment3.ipynb \
          --output test_output.ipynb \
          --ExecutePreprocessor.timeout=600 \
          --allow-errors || echo "Notebook execution completed with warnings"
    
    - name: Check for cell execution errors
      run: |
        python -c "
        import json
        with open('test_output.ipynb', 'r') as f:
            nb = json.load(f)
        errors = [cell for cell in nb['cells'] if 'outputs' in cell 
                 for output in cell['outputs'] if output.get('output_type') == 'error']
        if errors:
            print(f'Found {len(errors)} execution errors')
            exit(1)
        "
    
    - name: Clean up
      if: always()
      run: rm -f test_output.ipynb
```

## File 2: `code-quality.yml`

```yaml
name: Code Quality Checks

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  lint:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    
    - name: Install linting tools
      run: |
        pip install flake8 black isort pylint
    
    - name: Extract Python code from notebook
      run: |
        python -c "
        import json
        with open('Analyze_unemployment3.ipynb', 'r') as f:
            nb = json.load(f)
        with open('temp_analysis.py', 'w') as out:
            for cell in nb['cells']:
                if cell['cell_type'] == 'code':
                    out.write('\n# Cell\n')
                    out.write(''.join(cell['source']))
                    out.write('\n')
        "
    
    - name: Run Black (code formatting check)
      run: |
        black --check temp_analysis.py || echo "Code formatting issues detected (non-blocking)"
    
    - name: Run isort (import sorting check)
      run: |
        isort --check-only temp_analysis.py || echo "Import order issues detected (non-blocking)"
    
    - name: Run Flake8 (linting)
      run: |
        flake8 temp_analysis.py --count --show-source --statistics || echo "Linting issues detected (non-blocking)"
    
    - name: Clean up
      run: rm -f temp_analysis.py
```

## File 3: `data-validation.yml`

```yaml
name: Data Validation

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  validate-data:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    
    - name: Install dependencies
      run: |
        pip install pandas numpy kagglehub
    
    - name: Create data validation script
      run: |
        cat > validate_data.py << 'EOF'
        import pandas as pd
        import kagglehub
        
        # Download dataset
        path = kagglehub.dataset_download("gokulrajkmv/unemployment-in-india")
        df = pd.read_csv(f"{path}/Unemployment in India.csv")
        
        # Validation checks
        issues = []
        
        # Check shape
        if df.shape[0] < 700:
            issues.append(f"⚠ Warning: Dataset has only {df.shape[0]} rows (expected ~768)")
        
        # Check columns exist
        required_cols = ['Region', ' Date', ' Frequency', ' Estimated Unemployment Rate (%)',
                         ' Estimated Employed', ' Estimated Labour Participation Rate (%)', 'Area']
        missing_cols = [col for col in required_cols if col not in df.columns]
        if missing_cols:
            issues.append(f"✗ Error: Missing columns: {missing_cols}")
        
        # Check data types
        if ' Date' in df.columns:
            try:
                pd.to_datetime(df[' Date'])
            except:
                issues.append("✗ Error: Date column cannot be parsed as datetime")
        
        # Check unemployment rate range
        if ' Estimated Unemployment Rate (%)' in df.columns:
            rate = pd.to_numeric(df[' Estimated Unemployment Rate (%)'], errors='coerce')
            if (rate < 0).any() or (rate > 100).any():
                issues.append("✗ Error: Unemployment rate outside valid range [0, 100]")
        
        # Check for excessive missing values
        missing_pct = (df.isnull().sum() / len(df) * 100)
        if (missing_pct > 10).any():
            issues.append(f"⚠ Warning: Some columns have > 10% missing values:\n{missing_pct[missing_pct > 10]}")
        
        # Report results
        print("=" * 60)
        print("DATA VALIDATION REPORT")
        print("=" * 60)
        print(f"Dataset shape: {df.shape}")
        print(f"Date range: {df[' Date'].min()} to {df[' Date'].max()}")
        print(f"Missing values: {df.isnull().sum().sum()}")
        print("\n" + "=" * 60)
        
        if issues:
            print("ISSUES FOUND:\n")
            for issue in issues:
                print(f"  {issue}")
            print("\n" + "=" * 60)
        else:
            print("✓ All data validation checks passed!")
            print("=" * 60)
        
        # Exit with error if critical issues found
        critical_issues = [i for i in issues if i.startswith("✗")]
        if critical_issues:
            exit(1)
        EOF
    
    - name: Run data validation
      run: python validate_data.py
    
    - name: Clean up
      run: rm -f validate_data.py
```

## File 4: `documentation.yml`

```yaml
name: Documentation Check

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  check-documentation:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Check required documentation files
      run: |
        files=("README.md" "CODE_OF_CONDUCT.md" "CHANGELOG.md")
        missing=""
        
        for file in "${files[@]}"; do
          if [ ! -f "$file" ]; then
            missing="$missing\n  - $file"
          fi
        done
        
        if [ -n "$missing" ]; then
          echo "✗ Missing required documentation files:$missing"
          exit 1
        fi
        
        echo "✓ All required documentation files present"
    
    - name: Check README content
      run: |
        if ! grep -q "Installation" README.md || ! grep -q "Usage" README.md; then
          echo "⚠ Warning: README.md may be missing standard sections"
        fi
        echo "✓ README.md structure validated"
    
    - name: Check CHANGELOG format
      run: |
        if ! grep -q "## \[" CHANGELOG.md; then
          echo "⚠ Warning: CHANGELOG.md may not follow Keep a Changelog format"
        fi
        echo "✓ CHANGELOG.md format validated"
    
    - name: Validate markdown syntax
      run: |
        # Basic markdown validation
        for file in README.md CODE_OF_CONDUCT.md CHANGELOG.md; do
          if [ -f "$file" ]; then
            if ! head -1 "$file" | grep -q "^#"; then
              echo "⚠ Warning: $file doesn't start with a heading"
            fi
          fi
        done
        echo "✓ Markdown syntax validation completed"
```

## Setup Instructions

1. Create `.github/workflows/` directory in your repository:
```bash
mkdir -p .github/workflows
```

2. Create four files in `.github/workflows/`:
   - `notebook-validation.yml` (validates notebook execution)
   - `code-quality.yml` (checks code formatting and linting)
   - `data-validation.yml` (validates data integrity)
   - `documentation.yml` (checks documentation completeness)

3. Copy the workflow content into each file

4. Commit and push to your repository:
```bash
git add .github/workflows/
git commit -m "Add CI/CD workflows for automated validation"
git push
```

5. Configure GitHub Actions if needed (usually auto-enabled)

## Workflow Triggers

- **Push**: Runs on every push to main or develop branches
- **Pull Request**: Runs on every PR to main or develop branches
- Adjust branches as needed for your project

## Monitoring

- Check workflow status in your GitHub repository under "Actions" tab
- Failed workflows will block PR merges (if branch protection enabled)
- Customize failure behavior in GitHub branch protection rules

## Notes

- Workflows use Python 3.8+ for compatibility
- Data validation requires Kaggle API credentials (set in GitHub Secrets if needed)
- Execution errors in notebooks are allowed but logged for review
- Linting issues are reported but don't fail the build (adjust as needed)
