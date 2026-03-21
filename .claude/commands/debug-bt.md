# Debug Test Script Generator

Generate clean, focused test scripts for verifying implementations.

## When Called

1. Look at conversation context to find the handoff document path (from /implement)
2. Extract the working folder from that path (e.g., `work/v1_users/` from `work/v1_users/12b_contacts_api.md`)
3. Generate test script in same folder as `check_<session>_<feature>.js` or `.py`

## Script Type Decision

**JavaScript (F12 console paste)** when:
- Testing REST API endpoints
- Testing frontend state/behavior
- Testing WebSocket connections
- Anything that needs browser auth cookies

**Python (terminal)** when:
- Testing backend modules directly
- Testing database operations
- Testing signal processing
- Testing without browser context

**Rust (cargo test)** when:
- Testing Rust crate/workspace code
- Testing message serialization/deserialization
- Testing protocol handling
- Testing service integration

## Output Principles

### JavaScript Style
```javascript
(async () => {
  const results = [];
  const test = (name, pass, detail = '') => results.push({ name, result: pass ? 'PASS' : 'FAIL', detail });

  // ... tests that call test() ...

  console.clear();
  console.log('%c Title ', 'background:#4CAF50;color:white;font-size:14px;padding:4px 8px;border-radius:4px');
  console.table(results);
  const passed = results.filter(r => r.result === 'PASS').length;
  console.log(`%c${passed}/${results.length} passed`, passed === results.length ? 'color:green;font-weight:bold' : 'color:red;font-weight:bold');
})();
```

### Python Style
```python
#!/usr/bin/env python3
"""Test: [description]"""
import sys
results = []

def test(name, condition, detail=''):
    results.append({'name': name, 'result': 'PASS' if condition else 'FAIL', 'detail': detail})

# ... tests that call test() ...

# Output
print('\n' + '='*50)
print(f' {"TITLE":^46} ')
print('='*50)
for r in results:
    icon = '✓' if r['result'] == 'PASS' else '✗'
    print(f"  {icon} {r['name']}: {r['detail']}")
passed = sum(1 for r in results if r['result'] == 'PASS')
print('='*50)
print(f"  {passed}/{len(results)} passed")
sys.exit(0 if passed == len(results) else 1)
```

### Rust Style

**Unit tests** go in the same file as the code being tested:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_feature_name() {
        // Arrange
        let input = ...;

        // Act
        let result = function_under_test(input);

        // Assert
        assert_eq!(result, expected);
    }

    #[test]
    fn test_serialization_roundtrip() {
        let msg = Message { ... };
        let json = serde_json::to_string(&msg).unwrap();
        let parsed: Message = serde_json::from_str(&json).unwrap();
        assert_eq!(msg, parsed);
    }
}
```

**Integration tests** go in `your-crate/tests/`:
```rust
// tests/integration_test.rs
use your_crate::*;

#[tokio::test]
async fn test_health_endpoint() {
    // Start server in background or use test client
    let response = reqwest::get("http://localhost:8000/health")
        .await
        .unwrap();
    assert_eq!(response.status(), 200);

    let body: serde_json::Value = response.json().await.unwrap();
    assert_eq!(body["status"], "ok");
}
```

**Run commands:**
```bash
cd your-crate && cargo test                    # All tests
cd your-crate && cargo test test_name          # Single test
cd your-crate && cargo test -- --nocapture     # With println output
```

## Error Handling

| Error | Action |
|-------|--------|
| No handoff document in context | Ask user for the handoff path |
| Handoff has no testable success criteria | Report "No testable criteria found" and stop — don't generate an empty test |
| Target file/module not found | Note in test as a skipped check with path that was expected |
| Memory file read fails (debugging-patterns.md, codebase-quirks.md) | Continue without — these inform test coverage but aren't required |
| Test script generation fails (syntax error in output) | Re-read the target code, regenerate |
| Working folder doesn't exist | Create it before writing the test script |

## Rules

1. **Derive path from context** - Use same folder as the handoff document
2. **No noise** - Clear console (JS) or minimal prints (Python)
3. **Table output** - Results in clean tabular format
4. **PASS/FAIL** - Binary results with detail string
5. **Summary line** - X/Y passed with color/icon
6. **Self-contained** - No external dependencies beyond standard lib
7. **Auth-aware** - JS uses `credentials: 'include'`, Python uses env vars if needed
8. **Rust conventions** - Unit tests in same file, integration tests in `tests/` dir
9. **Rust async** - Use `#[tokio::test]` for async test functions

## Execution

1. Find handoff document path from conversation context
2. Extract working folder from that path
3. Read known patterns to inform test coverage:
   - `.claude/rules/debugging-patterns.md` — failure modes to cover in generated tests
   - `.claude/rules/codebase-quirks.md` — import patterns and setup gotchas that affect test code
4. Decide JS, Python, or Rust based on what's being tested
5. Write focused tests for the success criteria from the handoff, incorporating known failure modes and quirks into test setup and assertions
6. Save to appropriate location:
   - JS/Python: `<working_folder>/check_<session>_<feature>.js` or `.py`
   - Rust unit tests: inline in `your-crate/src/<module>.rs`
   - Rust integration tests: `your-crate/tests/<feature>_test.rs`
7. Tell user how to run it

$ARGUMENTS - Optional: force "js", "python", or "rust", or specify what to test
