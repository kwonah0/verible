# CLAUDE.md - AI Assistant Guide for Verible

This document provides essential context for AI assistants working with the Verible codebase.

## Project Overview

Verible is a SystemVerilog (IEEE 1800-2017) parser and developer toolchain. It provides:
- **Parser**: Handles unpreprocessed SystemVerilog source files
- **Style Linter** (`verible-verilog-lint`): Identifies style violations
- **Formatter** (`verible-verilog-format`): Manages whitespace and formatting
- **Language Server** (`verible-verilog-ls`): LSP implementation for editors
- **Other tools**: Syntax checker, diff, obfuscator, preprocessor, Kythe indexer

The codebase is written in C++17 and uses Bazel as the build system.

## Repository Structure

```
verible/
├── verible/
│   ├── common/           # Language-agnostic libraries
│   │   ├── analysis/     # Lint rule framework, syntax tree search
│   │   ├── formatting/   # Token partitioning, alignment
│   │   ├── lexer/        # Lexer interfaces and adapters
│   │   ├── lsp/          # Language Server Protocol implementation
│   │   ├── parser/       # Parser interfaces
│   │   ├── strings/      # String utilities
│   │   ├── text/         # Token, syntax tree, text structure classes
│   │   ├── tools/        # Patch tool
│   │   └── util/         # Generic utilities
│   └── verilog/          # SystemVerilog-specific code
│       ├── CST/          # Concrete Syntax Tree accessors
│       ├── analysis/     # Linter, file analysis
│       │   └── checkers/ # All lint rule implementations
│       ├── formatting/   # Formatter implementation
│       ├── parser/       # Lexer/parser (verilog.l, verilog.y)
│       ├── preprocessor/ # Preprocessing utilities
│       ├── tools/        # Command-line tools
│       │   ├── diff/
│       │   ├── formatter/
│       │   ├── kythe/
│       │   ├── lint/
│       │   ├── ls/
│       │   ├── obfuscator/
│       │   ├── preprocessor/
│       │   ├── project/
│       │   └── syntax/
│       └── transform/    # Code transformations
├── bazel/                # Bazel build rules (flex.bzl, bison.bzl)
├── doc/                  # Development documentation
├── .github/bin/          # Development helper scripts
├── third_party/          # Third-party code
└── external_libs/        # Library dependencies
```

## Build System

### Essential Commands

```bash
# Build all targets
bazel build -c opt //...

# Run all tests
bazel test -c opt //...

# Build and test specific target
bazel test //verible/verilog/tools/lint:lint-tool_test

# Build with static linking
bazel build -c opt --config=create_static_linked_executables //...

# Use local flex/bison
bazel build -c opt --//bazel:use_local_flex_bison //...
```

### Build Configuration

- **Language**: C++17 (supports up to C++23)
- **Dependencies**: Abseil, RE2, Protobuf, nlohmann/json, GoogleTest
- **Configuration**: `MODULE.bazel` (bzlmod), `.bazelrc`

### BUILD File Patterns

```python
load("@rules_cc//cc:defs.bzl", "cc_library", "cc_test", "cc_binary")

package(
    default_applicable_licenses = ["//:license"],
    default_visibility = ["//visibility:private"],
    features = ["layering_check"],
)

cc_library(
    name = "component",
    srcs = ["component.cc"],
    hdrs = ["component.h"],
    deps = [
        "//verible/common/util:logging",
        "@abseil-cpp//absl/strings",
    ],
)

cc_test(
    name = "component_test",
    srcs = ["component_test.cc"],
    deps = [":component", "@googletest//:gtest_main"],
)
```

## Code Style and Conventions

### Style Guide

- Follows [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
- Enforced by `clang-format` with configuration in `.clang-format`
- Static analysis via `clang-tidy` with configuration in `.clang-tidy`

### Formatting

Format code before commits:
```bash
clang-format --style=file -i <files>
```

Key formatting rules:
- Pointer/reference alignment: Right (`Type *ptr`, `Type &ref`)
- Based on Google style with minor customizations

### Naming Conventions

- **Files**: `kebab-case.h`, `kebab-case.cc`, `kebab-case_test.cc`
- **Classes**: `PascalCase`
- **Functions**: `PascalCase`
- **Variables**: `snake_case`
- **Constants**: `kPascalCase`
- **Namespaces**: `snake_case`

### Header Guards

```cpp
#ifndef VERIBLE_PATH_TO_FILE_H_
#define VERIBLE_PATH_TO_FILE_H_
...
#endif  // VERIBLE_PATH_TO_FILE_H_
```

## Testing

### Test Patterns

Every `.cc` file should have a corresponding `_test.cc`:
- Unit tests use Google Test (gtest)
- Test files follow naming: `component_test.cc` for `component.cc`
- Include both positive and negative test cases

### Running Tests

```bash
# All tests
bazel test -c opt //...

# Specific test
bazel test //verible/verilog/analysis/checkers:line-length-rule_test

# With coverage
MODE=coverage .github/bin/build-and-test.sh //specific:test
.github/bin/generate-coverage-html.sh
```

### Lint Rule Test Pattern

```cpp
TEST(RuleNameTest, Various) {
  const std::initializer_list<LintTestCase> kTestCases = {
    {"valid code without violations"},
    {"code with ", {kSymbolType, "violation"}, " here"},
  };
  RunLintTestCases<VerilogAnalyzer, RuleName>(kTestCases);
}
```

## Development Workflow

### Pre-Submit Checks

Run before creating a pull request:
```bash
.github/bin/before-submit.sh
```

This runs:
1. Compilation database refresh
2. Build cleaner (dependency verification)
3. All tests (including C++20, C++23)
4. clang-tidy analysis
5. Code formatting check
6. Potential problem detection

### Development Scripts

| Script | Purpose |
|--------|---------|
| `.github/bin/before-submit.sh` | Full pre-submit check |
| `.github/bin/run-format.sh` | Check/apply formatting |
| `.github/bin/make-compilation-db.sh` | Generate compile_commands.json for clangd |
| `.github/bin/run-clang-tidy-cached.cc` | Run clang-tidy with caching |
| `.github/bin/run-build-cleaner.sh` | Verify BUILD dependencies |
| `.github/bin/generate-coverage-html.sh` | Generate coverage report |
| `.github/bin/simple-install.sh` | Install binaries |

## Key Architectural Concepts

### Lexer/Parser Design

The lexer and parser are **decoupled**:
1. Lexer produces token stream (Flex-generated)
2. Optional transformation passes (contextualizer, filters)
3. Parser consumes tokens (Bison LALR(1))
4. Output: Concrete Syntax Tree (CST)

Key files:
- `verible/verilog/parser/verilog.l` - Flex lexer grammar
- `verible/verilog/parser/verilog.y` - Bison parser grammar

### Concrete Syntax Tree (CST)

- Captures ALL tokens (no information loss)
- Node types defined in `verible/verilog/CST/verilog_nonterminals.h`
- Accessor functions follow `GetXFromY` pattern
- Token enumerations are stable; node enumerations may change

### Lint Rule Types

Four analysis levels (in `verible/common/analysis/`):

1. **LineLintRule**: Line-by-line analysis
2. **TokenStreamLintRule**: Token-by-token analysis
3. **SyntaxTreeLintRule**: AST-based analysis (most common)
4. **TextStructureLintRule**: Multi-form analysis

### Adding a Lint Rule

1. Create files in `verible/verilog/analysis/checkers/`:
   - `rule-name-rule.h`
   - `rule-name-rule.cc`
   - `rule-name-rule_test.cc`

2. Inherit from appropriate base class:
```cpp
class RuleNameRule : public verible::SyntaxTreeLintRule {
 public:
  using rule_type = verible::SyntaxTreeLintRule;
  static const LintRuleDescriptor &GetDescriptor();
  void HandleNode(const verible::SyntaxTreeNode &node,
                  const verible::SyntaxTreeContext &context) final;
  verible::LintRuleStatus Report() const final;
 private:
  std::set<verible::LintViolation> violations_;
};
```

3. Register rule using `VERILOG_REGISTER_LINT_RULE` macro
4. Add to BUILD file
5. Write comprehensive tests

## Key Files Reference

### Core Interfaces
- `verible/common/analysis/lint-rule.h` - Lint rule base classes
- `verible/common/text/concrete-syntax-tree.h` - CST data structure
- `verible/common/text/token-info.h` - Token representation
- `verible/common/text/text-structure.h` - Unified text view

### Tool Entry Points
- `verible/verilog/tools/lint/verilog-lint.cc`
- `verible/verilog/tools/formatter/verilog-format.cc`
- `verible/verilog/tools/syntax/verilog-syntax.cc`
- `verible/verilog/tools/ls/verilog-ls.cc`

### Configuration
- `verible/verilog/analysis/verilog-linter-configuration.h`
- `verible/verilog/formatting/format-style.h`
- `verible/verilog/analysis/default-rules.h`

## Common Tasks

### Debugging Parser Issues

Use the syntax tool to examine parse trees:
```bash
bazel-bin/verible/verilog/tools/syntax/verible-verilog-syntax --printtree file.sv
```

### Understanding a Tool

1. Read `verible/verilog/tools/<tool>/README.md`
2. Review main binary: `<tool>.cc`
3. Check tests: `*_test.cc` or `*_test.sh`
4. Look at related libraries in `verible/verilog/analysis/`

### Modifying Parser/Lexer

1. Edit `verible/verilog/parser/verilog.l` (lexer)
2. Edit `verible/verilog/parser/verilog.y` (parser)
3. Update CST enums if needed
4. Run tests: `bazel test //verible/verilog/parser:...`

## Documentation Resources

- `doc/development.md` - Development getting started
- `doc/parser_design.md` - Lexer/parser architecture
- `doc/style_lint.md` - Lint rule development guide
- `doc/formatter.md` - Formatter internals
- `doc/indexing.md` - Kythe indexing

## CI/CD

GitHub Actions workflow (`.github/workflows/verible-ci.yml`) runs:
- Format and build cleaner checks
- clang-tidy static analysis
- Multi-mode testing (opt, asan, various C++ standards)
- Cross-compilation (x86_64, arm64)
- macOS builds
- Kythe indexing

## Tips for AI Assistants

1. **Always read files before modifying** - Understand existing patterns
2. **Check for corresponding test files** - Every `.cc` has a `_test.cc`
3. **Use existing patterns** - Follow conventions from similar files
4. **Run tests after changes** - `bazel test //path/to:target_test`
5. **Each directory has README.md** - Check for local documentation
6. **CST node enums are fragile** - Prefer stable token enums when possible
7. **Prefer accessor functions** - Use `GetXFromY` over direct tree traversal
8. **Include negative tests** - Test cases that should NOT trigger violations

## License

Apache 2.0 - All contributions require signing the Google CLA.
