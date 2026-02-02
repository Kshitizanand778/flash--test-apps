# flash--test-apps

# Test Impact Analyzer

A Python CLI tool for analyzing the impact of commits on Playwright test suites. Given a commit SHA, it identifies which tests were added, removed, or modified, including indirect impacts from helper method changes.

## Features

- **Direct Test Changes**: Detects tests that were added, removed, or modified directly
- **Indirect Changes**: Identifies tests that were impacted by helper/utility file modifications
- **Multiple Output Formats**: Text (human-readable) or JSON output
- **Git Integration**: Uses GitPython to analyze commits without modifying the repository
- **Smart Test Detection**: Parses Playwright test files to extract test names and understand dependencies

## Installation

### Requirements
- **Python 3.9 or higher**
- **pip** (comes with Python)

### Setup

```bash
# Create virtual environment (optional but recommended)
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

## Usage

### Command 1: Analyze with full options
```bash
python src/cli.py analyze --commit <SHA> --repo <REPO_PATH> [--format json|text]
```

### Command 2: Shorthand version
```bash
python src/cli.py list <REPO_PATH> <COMMIT_SHA>
```

### Examples

**Analyze commit 45433fd in flash-tests (text format):**
```bash
python src/cli.py analyze --commit 45433fd --repo ./flash-tests
```

**Get JSON output:**
```bash
python src/cli.py analyze --commit 45433fd --repo ./flash-tests --format json
```

**Shorthand:**
```bash
python src/cli.py list ./flash-tests 45433fd
```

## How It Works

### Analysis Steps

1. **Git Analysis**: Uses `git show` to get the full list of changed files in the commit
2. **File Classification**: Categorizes files as:
   - **Test files**: `*.spec.ts`, `*.test.ts`, etc.
   - **Helper files**: Files in `/helpers`, `/utils`, `/fixtures` directories or similarly named
   - **Other files**: Configuration, docs, etc.

3. **Direct Impact**: For test files directly modified:
   - **Added**: Extracts test names from the new file
   - **Removed**: Attempts to get test names before deletion
   - **Modified**: Uses diff analysis to identify which specific tests were changed

4. **Indirect Impact**: For helper files modified:
   - Searches all test files to find imports
   - Marks those tests as "modified" with `isIndirect: true`

5. **Output**: Returns a structured list of impacted tests with modification types

### Output Example (Text Format)

```
================================================================================
✨ COMMIT: 45433fd
================================================================================

📝 ADDED TESTS:
  ✓ "safeBash tool execution to get commit SHA"
    in tests/tool-execution/session.spec.ts

🔄 MODIFIED TESTS:
  ~ "Subscribe to session and verify in Subscribed sessions list"
    in tests/sessions.spec.ts
  ~ "Filter sessions list by users"
    in tests/sessions.spec.ts
  ~ "set human triage for failed test" (via helper change)
    in tests/test-runs.spec.ts
  ~ "another test using helper" (via helper change)
    in tests/test-runs.spec.ts

────────────────────────────────────────────────────────────────────────────────
📊 Summary: 1 test added, 4 tests modified (2 indirectly via helper changes)
================================================================================
```

### Output Example (JSON Format)

```json
{
  "commitSha": "45433fd",
  "impactedTests": [
    {
      "testName": "safeBash tool execution to get commit SHA",
      "filePath": "tests/tool-execution/session.spec.ts",
      "modificationType": "added"
    },
    {
      "testName": "set human triage for failed test",
      "filePath": "tests/test-runs.spec.ts",
      "modificationType": "modified",
      "isIndirect": true
    }
  ],
  "summary": "1 test added, 1 test modified (1 indirectly via helper changes)"
}
```

## Architecture

### `analyzer.py`
Core analysis logic with the `TestImpactAnalyzer` class:
- `analyze_commit(sha)`: Main entry point
- `_parse_commit_diff()`: Extracts changed files using GitPython
- `_extract_tests_from_file()`: Parses test names from files using regex
- `_find_tests_importing_file()`: Finds dependent tests
- `_refine_modified_tests()`: Provides detailed modification analysis

### `cli.py`
Command-line interface using Click:
- Handles argument parsing with Click decorators
- Formats and displays output with click utilities
- Provides two command variants (analyze and list) for flexibility

## Key Design Decisions

1. **AST-Free Parsing**: Uses regex to extract test names for simplicity and speed. Works well for standard Playwright test patterns.

2. **Helper Detection**: Uses common naming patterns and directory structures rather than complex import analysis.

3. **Graceful Degradation**: If detailed diff refinement fails, falls back to basic test extraction.

4. **Relative Paths**: All paths in output are relative to repository root for readability.

## Limitations & Future Improvements

1. **Test Name Extraction**: Uses regex - may miss tests with unusual formatting
   - Could improve with AST parsing using `@babel/parser`

2. **Helper Detection**: Pattern-based approach may miss some helper files
   - Could improve with full import graph analysis

3. **Performance**: Searches linearly through all test files
   - Could cache with a test index for large repos

4. **Cross-file Tests**: Doesn't handle test definitions spanning files
   - Rare in practice, could add special handling

## Development

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Run in development mode
python src/cli.py analyze --commit <SHA> --repo <PATH>

# Or use shorthand
python src/cli.py list <PATH> <SHA>
```

## Testing

To test with real commits, clone [flash-tests](https://github.com/empirical-run/flash-tests):

```bash
git clone https://github.com/empirical-run/flash-tests.git
python src/cli.py analyze --commit 45433fd --repo ./flash-tests
```

## License

MIT
