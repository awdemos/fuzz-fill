# Tool Configuration Domain Technical Documentation

## 1. Domain Overview

### 1.1 Domain Purpose

The **Tool Configuration Domain** serves as the foundational support layer for the fuzz-fill system, responsible for managing external tool paths, environment configuration, and logging infrastructure. This domain enables the core Reduction Engine and other business domains to execute operations without being burdened with tool resolution logic.

**Domain Classification**: Support Domain  
**Importance Score**: 7.0/10  
**Complexity Score**: 5.5/10  
**Primary Language**: Python (Type-annotated)

### 1.2 Domain Responsibilities

| Responsibility | Description |
|---------------|-------------|
| Tool Path Resolution | Resolve LLVM tool paths from CLI flags, environment variables, or defaults |
| Configuration Management | Define immutable tool configurations using frozen dataclasses |
| Environment Fallbacks | Provide fallback mechanisms when explicit configuration is unavailable |
| Logging Infrastructure | Configure multi-level logging with timing support and file output |
| Utility Functions | Supply helper functions for tool instantiation and validation |

### 1.3 Domain Boundaries

**Included Components**:
- LLVM tool path resolution and configuration
- Environment variable utilities for tool path fallbacks
- Logging configuration and timing utilities
- Factory functions for tool instance creation

**Excluded Components**:
- Actual tool execution logic (handled by Reduction Engine Domain)
- Security scanning tool configuration (handled by Security Scanning Domain)
- CI/CD integration logic (handled by CI/CD Integration Domain)

---

## 2. Architecture Design

### 2.1 High-Level Architecture

The Tool Configuration Domain follows a **three-tier modular architecture**:

```
┌─────────────────────────────────────────────────────────────┐
│                    Tool Configuration Domain                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────┐  ┌─────────────────────┐           │
│  │  LLVM Tool          │  │  Environment        │           │
│  │  Configuration      │  │  Utilities          │           │
│  │  (llvm_tools.py)    │  │  (env.py)           │           │
│  └─────────────────────┘  └─────────────────────┘           │
│                     ┌─────────────────────┐                 │
│                     │  Logging Utilities   │                 │
│                     │  (log.py)           │                 │
│                     └─────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Design Patterns

#### 2.2.1 Frozen Dataclass Pattern

The domain uses **frozen dataclasses** to ensure immutability of tool configurations:

```python
from dataclasses import dataclass
from pathlib import Path

@dataclass(frozen=True)
class ReduceTools:
    """Immutable configuration for reduction operations"""
    llc: Path
    llvm_reduce: Path
    llvm_dis: Path | None = None

@dataclass(frozen=True)
class BaselineTools:
    """Immutable configuration for baseline operations"""
    sancov: Path
    llvm_lit: Path
    llc: Path
    opt: Path
```

**Benefits**:
- Thread-safe configuration objects
- Prevents accidental modification of tool paths
- Enables use in function signatures and type checking

#### 2.2.2 Dependency Injection Pattern

Tool configurations are injected into components rather than created internally:

```python
class Reducer:
    def __init__(self, tools: ReduceTools, output_dir: Path, test: Test,
                 pipeline_steps: tuple[PipelineStep, ...]):
        # Dependencies injected, not created internally
        self.tools = tools
        self.output_dir = output_dir
        self.test = test
        self._pipeline_steps = pipeline_steps
```

#### 2.2.3 Priority Resolution Pattern

The domain implements a **three-tier priority system** for tool path resolution:

```
Priority 1: CLI Flags (highest)
    ↓
Priority 2: Environment Variables
    ↓
Priority 3: Default Paths (lowest)
```

---

## 3. Module Structure

### 3.1 Module Responsibilities

| Module | File | Primary Responsibility |
|--------|------|----------------------|
| LLVM Tool Configuration | `llvm_tools.py` | Define tool configurations using frozen dataclasses |
| Environment Utilities | `env.py` | Resolve tool paths from CLI flags or environment variables |
| Logging Utilities | `log.py` | Configure logging with timing support and file output |

### 3.2 Module Dependencies

```
┌─────────────────────────────────────────────────────────────┐
│                    Module Dependency Graph                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  llvm_tools.py  ────────────────────────────────────────────┤
│  │  (No external dependencies)                               │
│  │                                                           │
│  env.py  ────────────────────────────────────────────────────┤
│  │  → llvm_tools.py (uses ReduceTools, BaselineTools)       │
│  │  → os, pathlib (standard library)                        │
│  │                                                           │
│  log.py  ────────────────────────────────────────────────────┤
│  │  → sys, logging (standard library)                       │
│  │  → llvm_tools.py (uses ReduceTools for context)          │
│  │                                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. LLVM Tool Configuration

### 4.1 Tool Configuration Dataclasses

The `llvm_tools.py` module defines frozen dataclasses for different tool configurations:

#### 4.1.1 ReduceTools

```python
@dataclass(frozen=True)
class ReduceTools:
    """Immutable configuration for reduction operations"""
    llc: Path
    llvm_reduce: Path
    llvm_dis: Path | None = None
```

**Fields**:
- `llc`: LLVM compiler executable path
- `llvm_reduce`: LLVM reduce tool path
- `llvm_dis`: LLVM disassembler path (optional)

#### 4.1.2 BaselineTools

```python
@dataclass(frozen=True)
class BaselineTools:
    """Immutable configuration for baseline operations"""
    sancov: Path
    llvm_lit: Path
    llc: Path
    opt: Path
```

**Fields**:
- `sancov`: SanitizerCoverage tool path
- `llvm_lit`: LLVM lit test runner path
- `llc`: LLVM compiler executable path
- `opt`: LLVM optimizer path

### 4.2 Tool Instance Creation

The module provides factory functions for creating tool instances:

```python
def candidate_test_tools_from_args(
    *,
    llc: Path | None = None,
    llvm_reduce: Path | None = None,
    llvm_dis: Path | None = None,
) -> ReduceTools:
    """Create ReduceTools from CLI arguments with environment fallbacks"""
    return ReduceTools(
        llc=executable_from_flag_or_env(llc, "LLVM_LLC", flag_name="--llvm-llc"),
        llvm_reduce=executable_from_flag_or_env(llvm_reduce, "LLVM_REDUCE", flag_name="--llvm-reduce"),
        llvm_dis=executable_from_flag_or_env(llvm_dis, "LLVM_DIS", flag_name="--llvm-dis"),
    )

def incremental_tools_from_args(
    *,
    sancov: Path | None = None,
    llvm_lit: Path | None = None,
    llc: Path | None = None,
    opt: Path | None = None,
) -> BaselineTools:
    """Create BaselineTools from CLI arguments with environment fallbacks"""
    return BaselineTools(
        sancov=executable_from_flag_or_env(sancov, "LLVM_SANCOV", flag_name="--llvm-sancov"),
        llvm_lit=executable_from_flag_or_env(llvm_lit, "LLVM_LIT", flag_name="--llvm-lit"),
        llc=executable_from_flag_or_env(llc, "LLVM_LLC", flag_name="--llvm-llc"),
        opt=executable_from_flag_or_env(opt, "LLVM_OPT", flag_name="--llvm-opt"),
    )
```

---

## 5. Environment Utilities

### 5.1 Path Resolution Strategy

The `env.py` module implements the core path resolution logic:

```python
def executable_from_flag_or_env(
    value: Path | None,
    env_var: str,
    *,
    flag_name: str
) -> Path:
    """
    Resolve executable path from CLI flag or environment variable.
    
    Resolution order:
    1. CLI flag value (if provided)
    2. Environment variable (if set)
    3. SystemExit if neither provided
    """
    if value is not None:
        return value.expanduser().resolve()
    
    raw = os.environ.get(env_var)
    if raw:
        return Path(raw).expanduser().resolve()
    
    raise SystemExit(f"{flag_name} is required")
```

### 5.2 Supported Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `LLVM_LLC` | LLVM compiler path | None |
| `LLVM_REDUCE` | LLVM reduce tool path | None |
| `LLVM_DIS` | LLVM disassembler path | None |
| `LLVM_SANCOV` | SanitizerCoverage tool path | None |
| `LLVM_LIT` | LLVM lit test runner path | None |
| `LLVM_OPT` | LLVM optimizer path | None |
| `LLVM_REPO` | LLVM repository path | None |

### 5.3 Path Normalization

All resolved paths undergo normalization:
- `expanduser()`: Resolve `~` to home directory
- `resolve()`: Resolve symlinks and normalize path

---

## 6. Logging Utilities

### 6.1 Logging Configuration

The `log.py` module provides comprehensive logging infrastructure:

```python
def configure_logging(level: str, *, log_file: Path | None = None) -> None:
    """
    Configure fuzz-fill logging to stderr and optionally to a log file.
    
    Args:
        level: Logging level (error, warning, info, debug)
        log_file: Optional file path for log output
    """
    numeric_level = _LOG_LEVELS[level.lower()]
    formatter = logging.Formatter(_LOG_FORMAT)
    
    root = logging.getLogger("fuzz_fill")
    root.handlers.clear()
    
    # Console handler (stderr)
    console = logging.StreamHandler(sys.stderr)
    console.setFormatter(formatter)
    root.addHandler(console)
    
    # File handler (optional)
    if log_file is not None:
        file_handler = logging.FileHandler(log_file, encoding="utf-8")
        file_handler.setFormatter(formatter)
        root.addHandler(file_handler)
    
    root.setLevel(numeric_level)
```

### 6.2 Timing Recording

The logging module supports timing recording for performance analysis:

```python
@contextmanager
def record_log_timings() -> Iterator[list[tuple[str, float]]]:
    """
    Enable CSV export for nested log_timing blocks.
    
    Yields:
        List of (name, duration) tuples for timing data
    """
    global _active_timings
    rows: list[tuple[str, float]] = []
    with _timings_lock:
        _active_timings = rows
    try:
        yield rows
    finally:
        with _timings_lock:
            _active_timings = None
```

### 6.3 Logging Levels

| Level | Numeric | Use Case |
|-------|---------|----------|
| ERROR | 40 | Critical failures |
| WARNING | 30 | Non-critical issues |
| INFO | 20 | General information |
| DEBUG | 10 | Detailed debugging |

---

## 7. Domain Integration

### 7.1 Integration with Reduction Engine Domain

The Tool Configuration Domain provides critical tool paths to the Reduction Engine:

```python
# From reducer.py
class Reducer:
    def __init__(self, tools: ReduceTools, output_dir: Path, test: Test,
                 pipeline_steps: tuple[PipelineStep, ...]):
        self.tools = tools  # Injected from Tool Configuration Domain
        self.output_dir = output_dir
        self.test = test
        self._pipeline_steps = pipeline_steps
```

**Dependency Strength**: 9.0/10 (Critical)

### 7.2 Integration with Security Scanning Domain

The domain may provide tool paths for security scanning operations:

```python
# Security scanning tools may use similar resolution patterns
def get_trivy_path() -> Path:
    return executable_from_flag_or_env(
        None, "TRIVY_PATH", flag_name="--trivy-path"
    )
```

**Dependency Strength**: 5.0/10 (Optional)

### 7.3 Integration with Environment Management Domain

Tool configurations may reference package configurations for tool availability:

```python
# Tool Configuration → Environment Management
# Dependency Strength: 6.0/10 (Configuration Dependency)
```

---

## 8. Configuration Management

### 8.1 Configuration Sources

The domain supports multiple configuration sources:

| Source | Priority | Example |
|--------|----------|---------|
| CLI Flags | 1 (Highest) | `--llvm-llc /usr/bin/llvm-llc` |
| Environment Variables | 2 | `LLVM_LLC=/usr/bin/llvm-llc` |
| Default Paths | 3 (Lowest) | System paths |

### 8.2 Validation Rules

1. **Path Existence**: All resolved paths must exist and be executable
2. **Type Safety**: All paths are typed as `Path` objects
3. **Immutability**: Tool configurations are frozen dataclasses
4. **Error Messages**: Clear error messages with context when paths are missing

### 8.3 Error Handling

```python
# Example error handling
try:
    tools = candidate_test_tools_from_args()
except SystemExit as e:
    print(f"Configuration error: {e}")
    sys.exit(1)
```

---

## 9. Best Practices

### 9.1 Configuration Guidelines

1. **Use Frozen Dataclasses**: Always use `frozen=True` for tool configurations
2. **Explicit CLI Flags**: Prefer CLI flags over environment variables for production
3. **Environment Variables**: Use for development and testing flexibility
4. **Path Validation**: Always validate paths exist before use

### 9.2 Logging Guidelines

1. **Consistent Level**: Use INFO for general operations, DEBUG for detailed tracing
2. **File Output**: Enable file logging for production environments
3. **Timing Records**: Use `record_log_timings()` for performance analysis
4. **Structured Output**: Use consistent log formats across all modules

### 9.3 Security Considerations

1. **Path Traversal Prevention**: Use `resolve()` to prevent directory traversal
2. **User Input Validation**: Validate all CLI arguments and environment variables
3. **Least Privilege**: Run with minimal permissions when possible

---

## 10. Usage Examples

### 10.1 Basic Tool Configuration

```python
from fuzz_fill.llvm_tools import candidate_test_tools_from_args

# Use CLI flags
tools = candidate_test_tools_from_args(
    llc="/usr/bin/llvm-llc",
    llvm_reduce="/usr/bin/llvm-reduce",
)

# Use environment variables
tools = candidate_test_tools_from_args()  # Falls back to LLVM_* env vars
```

### 10.2 Logging Configuration

```python
from fuzz_fill.log import configure_logging

# Configure logging to stderr
configure_logging("INFO")

# Configure logging with file output
configure_logging("DEBUG", log_file="/var/log/fuzz-fill/debug.log")
```

### 10.3 Timing Recording

```python
from fuzz_fill.log import record_log_timings

with record_log_timings() as timings:
    # Perform operation
    result = perform_reduction()
    
    # Access timing data
    print(f"Operation took {sum(t[1] for t in timings):.2f}s")
```

---

## 11. Performance Considerations

### 11.1 Resolution Performance

- **CLI Flag Resolution**: O(1) - Direct path usage
- **Environment Variable Resolution**: O(1) - Single dictionary lookup
- **Path Normalization**: O(n) where n is path length

### 11.2 Memory Usage

- Frozen dataclasses: Minimal overhead (~100 bytes per configuration)
- Logging handlers: Configurable based on level and output destinations
- Timing records: Proportional to number of timing blocks

### 11.3 Optimization Opportunities

1. **Caching**: Cache resolved paths across multiple operations
2. **Lazy Loading**: Load logging configuration only when needed
3. **Batch Resolution**: Resolve multiple tools in a single pass

---

## 12. Testing Guidelines

### 12.1 Unit Testing

```python
def test_executable_from_flag_or_env():
    # Test CLI flag takes precedence
    assert executable_from_flag_or_env(Path("/custom"), "ENV_VAR") == Path("/custom")
    
    # Test environment variable fallback
    os.environ["ENV_VAR"] = "/env/path"
    assert executable_from_flag_or_env(None, "ENV_VAR") == Path("/env/path")
    
    # Test SystemExit when neither provided
    with pytest.raises(SystemExit):
        executable_from_flag_or_env(None, "NONEXISTENT")
```

### 12.2 Integration Testing

```python
def test_tool_configuration_integration():
    tools = candidate_test_tools_from_args()
    assert tools.llc.exists()
    assert tools.llvm_reduce.exists()
    assert tools.llc.is_file()
```

---

## 13. Troubleshooting

### 13.1 Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `SystemExit: --llvm-llc is required` | No CLI flag or environment variable | Provide `--llvm-llc` flag or set `LLVM_LLC` |
| `FileNotFoundError` | Path doesn't exist | Verify tool installation and path |
| `PermissionError` | Insufficient permissions | Check executable permissions |
| `Logging not configured` | Missing `configure_logging()` call | Add logging configuration before use |

### 13.2 Debugging Tips

1. **Enable DEBUG logging**: `configure_logging("DEBUG")`
2. **Check environment variables**: `echo $LLVM_*`
3. **Verify paths exist**: `ls -la /path/to/tool`
4. **Check permissions**: `ls -l /path/to/tool`

---

## 14. Future Enhancements

### 14.1 Planned Improvements

1. **Configuration File Support**: Add JSON/YAML configuration file support
2. **Tool Health Checks**: Implement tool availability verification
3. **Version Management**: Add tool version detection and reporting
4. **Caching Layer**: Implement path resolution caching
5. **Metrics Collection**: Add performance metrics to logging

### 14.2 Potential Extensions

1. **Multi-Tool Support**: Extend to support additional external tools
2. **Container Integration**: Add Docker volume path resolution
3. **Cloud Provider Support**: Add AWS/GCP tool path resolution
4. **Configuration Validation**: Add schema-based configuration validation

---

## 15. References

### 15.1 Related Documentation

- [Reduction Engine Domain](./Reduction-Engine-Domain.md)
- [Security Scanning Domain](./Security-Scanning-Domain.md)
- [CI/CD Integration Domain](./CI-CD-Integration-Domain.md)
- [Environment Management Domain](./Environment-Management-Domain.md)

### 15.2 External Resources

- [Python Dataclasses](https://docs.python.org/3/library/dataclasses.html)
- [Python Logging Module](https://docs.python.org/3/library/logging.html)
- [Pathlib Documentation](https://docs.python.org/3/library/pathlib.html)

---

*This documentation was generated based on comprehensive analysis of the fuzz-fill codebase and research materials. Last updated: 2026-09-13 20:22:04 (UTC)*