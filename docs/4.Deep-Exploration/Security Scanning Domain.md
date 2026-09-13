# Security Scanning Domain Technical Documentation

## 1. Overview

The Security Scanning Domain is a comprehensive vulnerability detection and security analysis subsystem within the fuzz-fill infrastructure. It provides automated security scanning capabilities for dependencies, infrastructure as code (IaC), GitHub Actions workflows, and hardcoded secrets. The domain integrates four specialized security tools—Trivy, Zizmor, Bandit, and Gitleaks—to deliver multi-layered security coverage in CI/CD pipelines.

### 1.1 Business Value

The Security Scanning Domain delivers critical security assurance for fuzz testing infrastructure by:

- **Dependency Vulnerability Detection**: Identifying known CVEs and misconfigurations in third-party packages
- **Infrastructure as Code Security**: Scanning IaC files for security misconfigurations
- **Workflow Security Analysis**: Detecting security issues in GitHub Actions workflows
- **Secret Detection**: Finding hardcoded credentials and sensitive data in code
- **CI/CD Integration**: Seamless integration with GitHub Actions for automated security validation

### 1.2 Target Users

| User Role | Primary Needs |
|-----------|---------------|
| Fuzz Testing Engineers | Automated test case reduction, coverage gap identification |
| DevOps Engineers | GitHub Actions integration, Docker image management |
| Security Engineers | Dependency vulnerability scanning, secret detection, workflow security linting |

---

## 2. Architecture

### 2.1 Domain Structure

The Security Scanning Domain follows a modular architecture with four specialized scanner components:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Security Scanning Domain                      │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Trivy      │  │    Zizmor    │  │    Bandit    │          │
│  │ Dependency   │  │ Workflow     │  │ Static       │          │
│  │ & IaC        │  │ Linter       │  │ Analyzer     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │   Gitleaks   │  │   Report     │                            │
│  │ Secret       │  │   Generator  │                            │
│  │ Detector     │  │              │                            │
│  └──────────────┘  └──────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Responsibilities

| Component | Primary Function | Target Scope |
|-----------|-----------------|--------------|
| **Trivy** | Dependency and IaC vulnerability scanning | `go.mod`, `package.json`, Dockerfiles, Terraform, Kubernetes manifests |
| **Zizmor** | GitHub Actions workflow security linting | `.github/workflows/*.yml` |
| **Bandit** | Python static security analysis | `.py` files |
| **Gitleaks** | Hardcoded secret detection | All files in repository |

### 2.3 Integration Architecture

The Security Scanning Domain integrates with other domains through the following patterns:

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD Integration Domain                      │
│                    (github_actions_api.py)                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Service Call
                              │ (Strength: 8.0)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Security Scanning Domain                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Trivy │ Zizmor │ Bandit │ Gitleaks                     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Implementation Details

### 3.1 Trivy: Dependency and IaC Vulnerability Scanner

#### 3.1.1 Overview

Trivy is a comprehensive vulnerability scanner that detects security issues in container images, file systems, and source code. In the fuzz-fill context, it scans dependencies and infrastructure as code files.

#### 3.1.2 Key Features

- **Multiple Scanner Types**: Supports `vuln`, `misconfig`, `secret`, and `license` scanners
- **Format Support**: SARIF, JSON, table, CycloneDX, SPDX, GitHub formats
- **Subtree Scanning**: Scans only changed files for CI efficiency
- **Severity Filtering**: Configurable severity thresholds

#### 3.1.3 Implementation

**Entry Point**: `/var/home/a/wikiwork/fuzz-fill/clone/scripts/github_actions/scan_tools/trivy.py`

**Supported Formats**:
```python
_SUPPORTED_FORMATS = {
    "sarif": "sarif",
    "json": "json",
    "table": "txt",
    "cyclonedx": "cdx.json",
    "spdx-json": "spdx.json",
    "github": "github.json",
}
```

**Supported Scanners**:
```python
_SUPPORTED_SCANNERS = ("vuln", "misconfig", "secret", "license")
_DEFAULT_SCANNERS = "misconfig,vuln"  # secret omitted (gitleaks covers it)
```

**Exit Codes**:
| Code | Meaning |
|------|---------|
| 0 | Clean scan (no findings) |
| 1 | Findings detected |
| 2 | Error occurred |

#### 3.1.4 Usage Pattern

```bash
# Scan dependencies and IaC files
trivy fs --format sarif --severity HIGH --output report.sarif .

# Subtree scanning for changed files
trivy fs --format sarif --subtree . --output report.sarif .
```

---

### 3.2 Zizmor: GitHub Actions Workflow Security Linter

#### 3.2.1 Overview

Zizmor is a security linter specifically designed for GitHub Actions workflows. It analyzes workflow files for security misconfigurations and best practice violations.

#### 3.2.2 Key Features

- **Workflow-Specific Analysis**: Detects security issues unique to GitHub Actions
- **Severity-Based Findings**: Categorizes issues by severity level
- **SARIF Output**: Generates standardized reports for CI integration
- **Security Misconfiguration Detection**: Identifies dangerous workflow patterns

#### 3.2.3 Implementation

**Entry Point**: `/var/home/a/wikiwork/fuzz-fill/clone/scripts/github_actions/scan_tools/zizmor.py`

**Key Functions**:
- Workflow linting and security analysis
- Severity-based finding tallies
- SARIF report generation

#### 3.2.4 Usage Pattern

```bash
# Analyze GitHub Actions workflows
zizmor .github/workflows/

# Generate SARIF report
zizmor .github/workflows/ --output report.sarif
```

---

### 3.3 Bandit: Python Static Security Analyzer

#### 3.3.1 Overview

Bandit is a static analysis tool for Python code that detects security issues such as hardcoded passwords, weak cryptographic keys, and insecure function calls.

#### 3.3.2 Key Features

- **Security Issue Detection**: Identifies common Python security vulnerabilities
- **Multiple Report Formats**: SARIF and non-SARIF formats supported
- **Severity Filtering**: Configurable severity thresholds
- **Codebase Scanning**: Comprehensive repository scanning

#### 3.3.3 Implementation

**Entry Point**: `/var/home/a/wikiwork/fuzz-fill/clone/scripts/github_actions/scan_tools/bandit.py`

**Key Functions**:
- Static security analysis on Python code
- Security issue detection
- SARIF and non-SARIF report generation

#### 3.3.4 Usage Pattern

```bash
# Scan Python codebase
bandit -r .

# Generate SARIF report
bandit -r . -f sarif -o report.sarif

# Filter by severity
bandit -r . --severity HIGH,CRITICAL
```

---

### 3.4 Gitleaks: Secret Detection Scanner

#### 3.4.1 Overview

Gitleaks is a fast and sensitive secret detection tool that identifies hardcoded credentials, API keys, and other sensitive data in code repositories.

#### 3.4.2 Key Features

- **Multiple Secret Patterns**: Detects various types of secrets and credentials
- **Multiple Report Formats**: Supports multiple output formats
- **Exit Code Handling**: Clean runs and findings distinguished
- **Repository Scanning**: Comprehensive codebase scanning

#### 3.4.3 Implementation

**Entry Point**: `/var/home/a/wikiwork/fuzz-fill/clone/scripts/github_actions/scan_tools/gitleaks.py`

**Key Functions**:
- Hardcoded credential detection
- Multiple secret pattern detection
- Multiple report format support
- Exit code handling for clean runs and findings

#### 3.4.4 Usage Pattern

```bash
# Scan repository for secrets
gitleaks detect --source .

# Generate report
gitleaks detect --source . --report-format sarif --output report.sarif
```

---

## 4. Workflow Integration

### 4.1 Security Scanning Pipeline

The Security Scanning Domain follows a structured pipeline workflow:

```mermaid
graph LR
    A[Start Security Scan] --> B[Initialize GitHub Actions API]
    B --> C[Configure Authentication]
    C --> D[Execute Trivy Scan]
    D --> E[Scan Dependencies]
    E --> F[Scan IaC Files]
    F --> G[Generate Trivy Report]
    G --> H[Execute Zizmor Lint]
    H --> I[Analyze Workflows]
    I --> J[Generate Zizmor Report]
    J --> K[Execute Bandit Scan]
    K --> L[Static Analysis]
    L --> M[Generate Bandit Report]
    M --> N[Execute Gitleaks Scan]
    N --> O[Detect Secrets]
    O --> P[Generate Gitleaks Report]
    P --> Q[Consolidate Findings]
    Q --> R[End Security Scan]
```

### 4.2 Key Steps

| Step | Operation | Purpose |
|------|-----------|---------|
| 1 | Initialize GitHub Actions API | Set up GitHub Actions API client with authentication |
| 2 | Execute Trivy Scan | Scan dependencies and IaC files for vulnerabilities |
| 3 | Execute Zizmor Lint | Analyze GitHub Actions workflows for security misconfigurations |
| 4 | Execute Bandit Scan | Detect Python security issues through static analysis |
| 5 | Execute Gitleaks Scan | Detect hardcoded credentials and secrets in code |
| 6 | Consolidate Findings | Aggregate all security findings by severity and generate reports |

---

## 5. Configuration and Usage

### 5.1 CLI Argument Parsing

Each scanner implements a main entry point with CLI argument parsing using argparse:

```python
# Example argument structure
parser = argparse.ArgumentParser(description="Security Scanner")
parser.add_argument("--format", choices=["sarif", "json", "table"], default="sarif")
parser.add_argument("--severity", default="HIGH")
parser.add_argument("--output", default="report.sarif")
parser.add_argument("--subtree", action="store_true")
```

### 5.2 Configuration Validation

Each tool supports configuration validation before scan execution:

```python
# Configuration validation pattern
def validate_config(config: dict) -> bool:
    """Validate scanner configuration"""
    required_fields = ["format", "severity"]
    for field in required_fields:
        if field not in config:
            raise ValueError(f"Missing required field: {field}")
    return True
```

### 5.3 Binary Installation and Caching

The module implements binary installation and caching mechanisms:

```python
# Binary caching mechanism
def install_binary(scanner: str, cache_dir: Path) -> Path:
    """Install or cache security scanner binary"""
    cache_path = cache_dir / f"{scanner}.bin"
    if not cache_path.exists():
        download_and_install(scanner, cache_path)
    return cache_path
```

---

## 6. Report Generation

### 6.1 SARIF Format

All scanners support SARIF (Static Analysis Results Interchange Format) for standardized reporting:

```json
{
  "runs": [
    {
      "tool": {
        "driver": {
          "name": "trivy",
          "version": "0.45.0"
        }
      },
      "results": [
        {
          "ruleId": "CVE-2021-1234",
          "level": "high",
          "message": "Vulnerability detected"
        }
      ]
    }
  ]
}
```

### 6.2 Non-SARIF Formats

Scanners also support alternative output formats:

| Format | Use Case |
|--------|----------|
| JSON | Machine-readable analysis |
| Table | Human-readable summary |
| CycloneDX | Software Bill of Materials |
| SPDX | Package provenance |

---

## 7. Security Considerations

### 7.1 Authentication Security

The Security Scanning Domain integrates with GitHub Actions API with multiple authentication methods:

```python
class AuthMethod(Enum):
    GITHUB_TOKEN = auto()   # CI environment
    GH_CLI = auto()         # Local development
    UNAUTHENTICATED = auto()  # Fallback
```

**Authentication Flow**:
1. Check for GITHUB_TOKEN environment variable
2. Fall back to gh CLI if installed and authenticated
3. Use unauthenticated requests as last resort (rate limited)

### 7.2 URL Security

The system implements URL scheme validation to prevent unauthorized access:

```python
_ALLOWED_URL_SCHEMES = frozenset({"https"})

def gha_open_https_url(url: str, *, timeout: int, headers: Mapping[str, str] | None = None):
    """Open an HTTPS URL, rejecting file:// and other schemes"""
    parsed = urlparse(url)
    if parsed.scheme not in _ALLOWED_URL_SCHEMES:
        raise ValueError(f"Refusing to fetch URL with unsupported scheme {parsed.scheme!r}")
    return urlopen(Request(url, headers=headers or {}), timeout=timeout)
```

### 7.3 Exit Code Handling

Each scanner implements proper exit code handling:

| Scanner | Exit Code 0 | Exit Code 1 | Exit Code 2 |
|---------|-------------|-------------|-------------|
| Trivy | Clean scan | Findings detected | Error |
| Zizmor | Clean scan | Findings detected | Error |
| Bandit | Clean scan | Findings detected | Error |
| Gitleaks | Clean scan | Findings detected | Error |

---

## 8. Best Practices

### 8.1 CI/CD Integration

1. **Parallel Execution**: Run independent scanners (Trivy, Bandit, Gitleaks, Zizmor) in parallel to reduce total scan time
2. **Incremental Scanning**: Implement subtree scanning for changed files only in CI/CD pipelines
3. **Severity Filtering**: Configure severity thresholds appropriate for your security policy
4. **Report Consolidation**: Aggregate findings from all scanners into a unified report

### 8.2 Configuration Management

1. **Environment Variables**: Use environment variables for sensitive configuration
2. **Caching**: Implement binary caching to avoid redundant installations
3. **Validation**: Validate all configuration before scan execution
4. **Logging**: Enable comprehensive logging for troubleshooting

### 8.3 Security Hardening

1. **Authentication**: Always use authenticated API calls in CI/CD environments
2. **URL Validation**: Validate all external URLs before fetching
3. **Secret Detection**: Run Gitleaks as a blocking check in CI/CD pipelines
4. **Regular Updates**: Keep security scanner binaries up to date

---

## 9. Domain Relationships

### 9.1 Dependency Strengths

| Source Domain | Target Domain | Relation Type | Strength |
|---------------|---------------|---------------|----------|
| Reduction Engine | Tool Configuration | Configuration Dependency | 9.0 |
| Reduction Engine | Security Scanning | Service Call | 5.0 |
| CI/CD Integration | Security Scanning | Service Call | 8.0 |
| Environment Management | CI/CD Integration | Data Dependency | 7.0 |
| Tool Configuration | Environment Management | Configuration Dependency | 6.0 |
| Reduction Engine | Environment Management | Data Flow | 7.5 |

### 9.2 Critical Path

The **Reduction Engine Domain** has the highest dependency strength (9.0) on **Tool Configuration**, making tool path resolution a critical path item that must succeed before any reduction operations can proceed.

### 9.3 Cross-Domain Integration

The **CI/CD Integration Domain** acts as a central orchestrator, connecting Security Scanning and Environment Management domains, making it a key integration point for system-wide operations.

---

## 10. Optimization Opportunities

### 10.1 Performance Improvements

1. **Parallel Execution**: The security scanning workflow could benefit from parallel execution of independent scanners to reduce total scan time
2. **Caching Mechanisms**: Tool path resolution and package configurations could implement caching to avoid redundant lookups and installations
3. **Progressive Configuration Loading**: The reduction pipeline could implement lazy loading of reduction passes to reduce initial memory footprint

### 10.2 Incremental Scanning

The security scanning workflow could implement incremental scanning for changed files only, reducing scan time in CI/CD pipelines:

```python
# Subtree scanning for changed files
trivy fs --subtree . --format sarif --output report.sarif .
```

### 10.3 Tool Health Monitoring

Adding health checks for external tools (LLVM, Trivy, etc.) before pipeline execution could fail fast and provide clearer error messages.

---

## 11. Conclusion

The Security Scanning Domain provides comprehensive vulnerability detection and security analysis capabilities for the fuzz-fill infrastructure. By integrating Trivy, Zizmor, Bandit, and Gitleaks, the domain delivers multi-layered security coverage for dependencies, infrastructure as code, GitHub Actions workflows, and hardcoded secrets.

### Key Strengths

- **Clear layered architecture** with distinct scanner components
- **Comprehensive security integration** with multiple tools
- **Robust error handling** and validation
- **Flexible configuration management** with CLI and environment variable support
- **Well-documented Docker build process** for reproducible environments

### Areas for Improvement

- Enhanced inline documentation for complex reduction logic
- Expanded test coverage for security scanning tools
- Performance metrics collection
- Structured logging implementation

The Security Scanning Domain is a critical component of the fuzz-fill infrastructure, providing essential security assurance for fuzz testing pipelines and CI/CD workflows.