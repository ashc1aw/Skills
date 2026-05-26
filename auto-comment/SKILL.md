---
name: auto-comment
description: Keep comments in sync with code. When generating, modifying, or deleting code, automatically add, sync, or remove the corresponding comments. Also triggered when the user explicitly asks to add, update, or review comments.
version: 1.0.0
allowed-tools: Read, Edit, Write, Glob
---

# Auto Comment Skill

## Core Principles

1. **Necessity only** — comment WHY, not WHAT. Well-named code is self-documenting. A comment is only needed when the intent is non-obvious.
2. **Token efficient** — every comment costs tokens. Prefer short, information-dense comments. Never generate boilerplate or filler text.
3. **Sync with code** — when implementation changes, update or remove outdated comments. Stale comments are worse than none.
4. **Concise & accurate** — one line when possible, short block when necessary. No word wasted.
5. **Non-obvious test**: would a developer familiar with the language need more than ~3 seconds to grasp the *intent* (not the mechanics)? If yes, it needs a comment.

## What Needs a Comment

### Always (public API surface)
- Classes, exported functions, public methods → standard doc comment
- Parameters with non-obvious constraints (range, format, side effects)
- Return values that aren't deducible from the function name

### Only when non-obvious (internal logic)
- Complex conditionals (3+ intertwined conditions)
- Algorithm choice rationale (why this approach vs. the obvious one)
- Performance-sensitive paths with non-obvious tradeoffs
- Workarounds for external bugs or framework quirks
- Magic numbers that can't be named as constants

### Skip entirely
- Simple getters/setters/properties
- CRUD operations with clear names
- Obvious variable assignments
- Boilerplate (constructors, serializer fields, route registrations)
- Code that reads like pseudocode already

## Comment Format by Language

| Language | Standard | Inline | Block |
|----------|----------|--------|-------|
| TypeScript/JavaScript | JSDoc `/** ... */` | `// ...` | `/* ... */` |
| Python | Docstring `"""..."""` | `# ...` | `# ...` (multi-line `#`) |
| Go | Doc comment `// Name ...` | `// ...` | `/* ... */` |
| Rust | Doc comment `/// ...` | `// ...` | `/* ... */` |
| Java/Kotlin | Javadoc `/** ... */` | `// ...` | `/* ... */` |
| C# | XML doc `/// ...` | `// ...` | `/* ... */` |
| C/C++ | Doxygen `/** ... */` or `///` | `// ...` | `/* ... */` |

**Doc comment content rule**: One summary line. Add `@param`/`@returns`/`@throws` only when the parameter/return/exception is non-obvious. Skip `@type`, `@date`, `@since`, `@version` — these are noise.

```typescript
// Good — one line, says WHY
/** Converts raw bytes to Base64, preserving URL-safe padding for JWT payloads. */
function encodeUrlSafe(data: Uint8Array): string { ... }

// Bad — boilerplate, restates the signature
/**
 * Encodes data to Base64 URL-safe string.
 * @param data - The data to encode
 * @returns The encoded string
 */
function encodeUrlSafe(data: Uint8Array): string { ... }
```

```python
# Good — docstring for public API, inline for non-obvious logic
def retry_with_backoff(fn: Callable, max_attempts: int = 3) -> Any:
    """Call `fn` with exponential backoff, raising the last error if all attempts fail."""
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError:
            if attempt == max_attempts - 1:
                raise
            time.sleep(2 ** attempt)  # 1s, 2s, 4s — AWS rate-limit recovery window
```

**Chinese comment example** (when user prompts in Chinese or specifies `// language: zh`):

```python
# 中文场景 — 公开 API 用中文 docstring，非显而易见的逻辑用中文行内注释
def retry_with_backoff(fn: Callable, max_attempts: int = 3) -> Any:
    """调用 `fn` 并使用指数退避重试，所有尝试均失败则抛出最后一次错误。"""
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError:
            if attempt == max_attempts - 1:
                raise
            time.sleep(2 ** attempt)  # 1s, 2s, 4s — 匹配 AWS 限流恢复窗口
```

## Comment Language Detection

Determine comment language by priority:
0. **User-specified language** — when the user explicitly specifies a language (e.g., `// language: zh`, "用中文注释", "add comments in Chinese"), use it. This overrides all other signals.
1. **User's prompt language** — if the user writes in Chinese, output comments in Chinese; if in English, output English.
2. **Existing comments in the file** — match the language of surrounding comments for consistency.
3. **Project convention** — if the codebase has a clear comment-language pattern, follow it.

Default to English if no signal is present.

## Comment Sync (Update Mode)

When code implementation changes, review every comment within the changed scope:

1. **Stale** — comment describes behavior that no longer exists → **update or remove**
2. **Drifted** — comment is correct but code was refactored around it → **relocate to the right position**
3. **Still valid** — comment intent still matches the code → **leave as-is**

Sync trigger: user modifies a function/class body, or explicitly requests "update comments".

## Workflow

### Add Mode (default)
```
1. Identify the scope (selected code, file, or directory)
2. For each code element in scope:
   a. Is it a public API? → add concise doc comment (summary line only, ≤3 lines total)
   b. Does it contain non-obvious logic? → add minimal inline comment (≤1 line)
   c. Is it self-documenting? → skip
3. Output only the annotated sections (the lines/blocks where comments were added), not the entire file.

```

### Update Mode (user requests "sync comments" or code is modified)
```
1. Identify changed functions/classes/methods
2. Compare existing comments against current implementation
3. Update stale comments, remove obsolete ones, keep valid ones
4. Output only affected sections (not the entire file) to save tokens
```

## Anti-Patterns

- **Do NOT** comment every line or every function — that's noise, not documentation.
- **Do NOT** restate the function signature in prose — `@param name - the name` is worthless.
- **Do NOT** write multi-paragraph docstrings for trivial functions.
- **Do NOT** add comments in generated/output code (build artifacts, minified files).
- **Do NOT** use comments to "fix" unclear code — rename the variable/function instead.

## Token Budget Awareness

- A comment must carry more value than its token cost.
- Default max: **1 line** for inline, **2-3 lines** for doc comments.
- For large files, prioritize public API comments and defer internal-only comments unless asked.
- In update mode, return only changed comment blocks, not the full file.