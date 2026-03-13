---
name: nasa-coding-standards
description: >-
  Analyzes and refactors code to strictly comply with NASA JPL's "Power of 10"
  coding rules for safety-critical software. Supports C/C++ (original rules) and
  interpreted languages like Python, JavaScript, TypeScript, and Go (adapted rules).
  Use when user says "nasa", "/nasa", "power of 10", "nasa coding standards",
  "nasa review", "safety-critical code", "nasa rules", or asks to enforce NASA
  coding guidelines on their code. Do NOT use for general code review unrelated
  to NASA standards.
metadata:
  author: zaydk
  version: 1.0.0
  tags: [code-quality, nasa, safety-critical, power-of-10]
---

# NASA Coding Standards Enforcer

Enforce NASA JPL's "Power of 10" coding rules on any codebase. These rules were
created by Gerard Holzmann at NASA's Jet Propulsion Laboratory to produce
verifiable, reliable, safety-critical software.

## Workflow

### Step 1: Identify the Target Code

Ask the user which files or code they want audited. If they paste code directly,
use that. If they reference files, read them.

### Step 2: Detect Language Category

Determine which rule set applies:

- **C/C++**: Apply the original 10 NASA JPL rules. Consult `references/rules-c.md`.
- **Interpreted languages** (Python, JavaScript, TypeScript, Go, Java, Ruby, PHP, Rust, etc.): Apply the adapted rules. Consult `references/rules-interpreted.md`.
- **Mixed codebase**: Apply the appropriate rule set per file.

### Step 3: Audit -- Check Every Rule

Walk through each of the 10 rules against the code. For every violation found, record:

- **Rule number and name**
- **Location** (function name, line range, or code snippet)
- **What is wrong** (one sentence)
- **Severity**: CRITICAL (will cause runtime failure or undefined behavior), HIGH (likely bug or unmaintainable), MEDIUM (style/analyzability concern)

Be thorough. Check every function, every loop, every variable declaration.

### Step 4: Produce the Violation Report

Present findings as a structured table:

```
| # | Rule | Location | Violation | Severity |
|---|------|----------|-----------|----------|
| 1 | R2   | main():12 | while loop has no fixed upper bound | CRITICAL |
| 2 | R4   | processData():1-95 | Function is 95 lines, exceeds 60-line limit | HIGH |
```

Below the table, provide a summary:
- Total violations by severity
- Overall compliance percentage (rules passed / total checks)

### Step 5: Refactor the Code

Rewrite the code to resolve ALL violations. For each change:
- Preserve the original logic and behavior exactly
- Add a brief inline comment citing the rule ONLY where the fix is non-obvious (e.g., `// NASA R2: explicit loop bound`)
- Do not add comments that merely narrate what the code does

### Step 6: Explain Key Changes

After the refactored code, provide a short summary of the most impactful changes.
Group by rule number. Keep it concise -- focus on the "why", not the "what".

## Output Format

Structure every response as:

1. **Language detected**: [language]
2. **Rule set applied**: [Original NASA C rules / Adapted interpreted rules]
3. **Violation Report**: [table]
4. **Refactored Code**: [full code]
5. **Change Summary**: [grouped by rule]

## Important Guidelines

- Be strict. NASA rules exist to prevent catastrophic failures. Do not hand-wave
  violations or treat them as suggestions.
- When a rule is subjective (e.g., function length near the boundary), flag it
  with severity MEDIUM and let the user decide.
- If code already fully complies, say so explicitly and note which rules were
  checked.
- For large files, process function-by-function rather than trying to hold the
  entire file in working memory at once.
- If the user provides only a snippet, audit what is visible and note which rules
  cannot be fully checked without the broader context.

## Quick Reference -- The 10 Rules

For C/C++ (original):

1. Simple control flow only -- no goto, setjmp/longjmp, recursion
2. Fixed upper bounds on all loops
3. No dynamic memory allocation after init
4. Functions max 60 lines
5. Minimum 2 assertions per function
6. Data declared at smallest scope
7. Check all return values and validate all parameters
8. Preprocessor limited to includes and simple macros
9. Pointers limited to single dereference; no function pointers
10. Compile with all warnings; zero warnings allowed

For interpreted languages (adapted):

1. No goto equivalents; no recursion; avoid fall-through switch
2. All iteration must be provably bounded
3. Memory usage must be bounded (pagination, chunking, streaming)
4. Functions max 60 lines
5. Fail loudly -- assertions and exceptions, never silent failures
6. Smallest possible variable scope; no globals
7. Never silently discard errors or return values
8. Minimize transpiler/build complexity
9. Don't mutate arguments; use type safety features
10. Use all available linters and static analyzers; zero warnings

For full rules with code examples, consult the files in `references/`.
