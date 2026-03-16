# Test Coverage (JaCoCo)

> JaCoCo coverage analysis patterns and known false positives. Referenced by `/android:write-tests` during coverage maximization.

## Method-Level Analysis

Class-level coverage can hide uncovered methods. Always check per-method breakdown:

```python
# Parse JaCoCo XML for method-level coverage
import xml.etree.ElementTree as ET
tree = ET.parse('app/build/reports/jacoco/jacocoTestReport/jacocoTestReport.xml')
root = tree.getroot()

for cls in root.findall('.//class'):
    name = cls.get('name', '')
    if 'TargetClassName' in name:
        print(f'Class: {name}')
        for method in cls.findall('.//method'):
            mname = method.get('name')
            for c in method.findall('counter'):
                if c.get('type') == 'INSTRUCTION':
                    mi = int(c.get('missed', 0))
                    ci = int(c.get('covered', 0))
                    total = mi + ci
                    pct = (ci / total * 100) if total > 0 else 0
                    status = 'COVERED' if mi == 0 else 'PARTIAL' if ci > 0 else 'UNCOVERED'
                    print(f'  {status}: {mname} ({ci}/{total} = {pct:.0f}%)')
```

**Why method-level matters:** A class at 85% overall might have 3 methods at 0%. Those methods are the ones most likely to have bugs.

## JaCoCo 0.8.14+ Kotlin False Positives

JaCoCo 0.8.14+ fixes many Kotlin-specific false positives. If using an older version, these branches will show as uncovered but are NOT testable:

| False Positive | JaCoCo Version Fixed | Description |
|---------------|---------------------|-------------|
| Coroutine state machine | 0.8.14 | `suspend` functions compile to state machines with untestable branches |
| Elvis operator (`?:`) | 0.8.14 | Null check on non-nullable type generates dead branch |
| Safe call chain (`?.`) | 0.8.14 | `foo?.bar?.baz` generates intermediate null checks |
| Suspend lambda | 0.8.14 | Lambda passed to `withContext`/`async` generates extra branches |
| Inline value class | 0.8.14 | Boxing/unboxing generates branches |
| `when` exhaustive else | Not fixed | Sealed class `else ->` is dead code by definition |
| `data class` generated methods | Not fixed | `copy()`, `toString()`, `hashCode()`, `componentN()` |
| Compose pausable composition | Not fixed | Compose compiler generates restart groups |

## Do NOT Write Tests For

- **Unreachable `else` branches** on sealed/enum `when` expressions
- **Generated `data class` methods** (`copy`, `toString`, `hashCode`, `equals`, `componentN`)
- **Compose compiler-generated code** (restart groups, state management)
- **Kotlin null-check bytecode** on non-nullable types (JaCoCo artifact)
- **Hilt `Factory$DefaultImpls`** (generated DI code)

Document these exclusions in the coverage report:
```markdown
> **Unreachable (excluded):**
> - 2 branches: Kotlin null-check on non-nullable return (lines 45, 78)
> - 1 branch: sealed class exhaustive else (line 102)
> - 1 method: Hilt Factory$DefaultImpls (generated)
```

## Coverage Targets

| Class Type | Instructions | Branches | Lines | Detection |
|------------|-------------|----------|-------|-----------|
| Standard class | 95% | 90% | 95% | Default |
| Hardware ViewModel | 50% | 30% | 50% | Imports contain BLE/WiFi/Bluetooth classes, or class > 800 lines |
| Generated code | Skip | Skip | Skip | Hilt Factory, data class methods, Compose compiler |

> For `turbineScope` multiple-flow testing pattern, see `test-patterns.md`.
