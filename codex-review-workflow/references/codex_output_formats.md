# Codex CLI Output Format Variations

Codex CLI returns structured markdown output. **IMPORTANT: The format is highly variable** - Codex uses different section headers at different times. This document catalogs all observed format variations to enable robust parsing.

## Initial Review - Possible Output Formats

### Format 1: Categorized by severity
- **Bug** - Critical bugs or incorrect logic
- **Security** - Security vulnerabilities
- **Usability** - User experience issues
- **Maintainability** - Code quality issues
- **Suggestions** - Optional improvements

### Format 2: Grouped by priority
- **Key Issues** or **Key Findings** - All critical problems grouped together
- **Maintainability / Best Practices** - Combined quality improvements
- **Suggestions** - Optional improvements
- **Next Steps** or **Next steps** - Recommended actions (may appear with or without colon)

### Format 3: Mixed format
- **Key Issues** + **Suggestions**
- Individual issue types like **Bug**, **Usability**, etc.

### Format 4: Findings-based
- **Findings** - All issues listed together
- **Next Steps** - Recommended actions

### Format 5: Single issue format
- **Issue** (singular) - When only one problem is found
- Single dash-prefixed item with detailed explanation

### Format 6: Severity-prefixed bullet list
- No bold headers
- Plain bullet list with severity prefixes: "High:", "Medium:", "Low:"
- Each item starts with severity level followed by file:line reference

## Follow-up Review - Possible Output Formats

### Format 1: Detailed breakdown
- **Findings** - New issues discovered after fixes
- **Resolved Checks** - Confirmation that previous issues are fixed
- **Next Steps** - Recommended actions

### Format 2: Combined assessment
- **Review** - All feedback in one section (confirmed fixes + remaining issues)
- **Review Findings** - Overall assessment with ✅/❌ indicators

### Format 3: Structured verification
- **Resolved Checks** - Line-by-line confirmation of fixes
- **Findings** - Any remaining or new issues

### Format 4: Status checklist
- **Findings** - New or remaining issues
- **Status of the requested fixes** - Numbered list of original issues with ✅/❌ status indicators
  - Shows which fixes were successful and which remain outstanding

### Format 5: Plain text confirmation
- Opens with plain text: "All [N] issues look resolved—no further findings"
- No bold section headers
- Bullet points with file:line references confirming fixes
- May include "Residual risk" notes for edge cases

## Issue Structure

Each issue typically includes:
- File path and line number (e.g., `email_validator.py:19`)
- Description of the problem
- Why it's an issue
- Sometimes example fixes or code snippets

## Real-World Examples from Validated Reviews

### Initial Review - Example 1: Categorized by type
```
- **Bug** `email_validator_v2.py:58`: The domain pattern `[a-zA-Z0-9.-]+` accepts leading dots/hyphens and double dots, so inputs like `user@.example.com` or `user@domain..com` currently pass as "Valid email".
- **Bug** `email_validator_v2.py:42`: The max-length guard allows 320 chars, but RFC 5321/3696 limit email addresses to 254 characters.
- **Usability** `email_validator_v2.py:17` and `email_validator_v2.py:53`: Inputs with leading/trailing spaces are treated as invalid even if trimming would reveal a good address.
- **Maintainability** `email_validator_v2.py:20` and `email_validator_v2.py:58`: The same literal regex is redefined twice.
```

### Initial Review - Example 2: Grouped by priority (variant A)
```
**Key Issues**
- `email_validator_v3.py:18` and `email_validator_v3.py:54` reject perfectly valid addresses such as `user+tag@example.com` because the character class is limited to `\w`.
- `email_validator_v3.py:18` and `email_validator_v3.py:54` also accept invalid domains like `user@bad_domain.com` or `user@domain..com`.
- `email_validator_v3.py:20` and `email_validator_v3.py:55` call `re.match` without ensuring `email`/`email_string` is a string.
- `email_validator_v3.py:36` caps total length at 320, which is non-standard.

**Maintainability / Best Practices**
- `email_validator_v3.py:18` and `email_validator_v3.py:54` duplicate the regex; hoist a compiled constant.
- `email_validator_v3.py:32` should trim surrounding whitespace.

**Next Steps**
1. Decide whether to refine the home-grown regex or switch to a proven validator.
2. Add tests for edge cases mentioned above.
```

### Initial Review - Example 3: Grouped by priority (variant B)
```
**Key Findings**
- email_validator_v4.py:11 – The character class `[\w\.\-]+` bans valid local-part characters such as `+`, `'`, `%`, `/`, etc., so common addresses like `user+tag@example.com` incorrectly fail.
- email_validator_v4.py:11 – The same regex accepts clearly invalid domains: `\w` allows underscores, and the pattern does not prevent leading/trailing dots or hyphens.
- email_validator_v4.py:31 & :62 – `re.match` recompiles the raw pattern each call. Precompiling with `re.compile(...).fullmatch()` avoids mistakes if the pattern changes.
- email_validator_v4.py:34-65 – `validate_with_reason` duplicates logic from `is_valid` yet still cannot explain which rule inside the regex failed.

Next steps:
1. Rewrite the validation pipeline (or adopt a maintained email validation library) to cover the missing allowed characters and tighten domain rules.
2. Add unit tests for the problematic examples (`user+tag@example.com`, `user@_host.com`, `user@.example.com`) to lock in the fixes.
```

### Initial Review - Example 4: Single issue
```
**Issue**
- `email_validator_final.py:11` rejects domains containing consecutive hyphens (e.g. `user@co--op.org`, `user@xn--example.com`), even though those are legitimate per RFC rules. The domain regex requires every hyphen to be followed immediately by an alphanumeric, so `EMAIL_RE.fullmatch("user@co--op.org")` returns `False`. This violates requirement #3 ("supporting plus signs and complex formats") and should be relaxed, e.g. by allowing hyphens inside each label with something like `[a-zA-Z0-9](?:[a-zA-Z0-9-]*[a-zA-Z0-9])?`. Adding a test for a valid double-hyphen domain would also exercise this case.
```

### Initial Review - Example 5: Severity-prefixed list
```
- High: `email_validator.py:17-18` and `email_validator.py:34-35` use `[\w\.-]+` for the domain portion, which accepts underscores and dotted labels like `user@.example.com`; real DNS labels allow only letters/digits/hyphen and no leading/trailing dots. The regex also permits a single-character TLD via `\w+`, so `user@example.c` or `user@example._t` would be accepted. Tighten the pattern to something like `r"^(?P<local>...)@(?P<domain>(?:[A-Za-z0-9](?:[A-Za-z0-9-]{0,61}[A-Za-z0-9])?\.)+[A-Za-z]{2,})$"` and switch to `re.fullmatch` to avoid partial matches.
- Medium: `email_validator.py:31` enforces a 320-character ceiling but skips the RFC 5321 limits (local part ≤64, domain ≤255); additionally, `len(email_str)` explodes if `None` sneaks in. Guard with `if not isinstance(email_str, str)` and enforce both per-part limits after a `.strip()` to ignore accidental surrounding whitespace.
- Medium: `email_validator.py:17` / `34` reject many addresses you probably want to accept (e.g., `user+tag@example.com`, quoted locals, apostrophes). Decide whether you need RFC compliance or an "everyday" validator; if the latter, expand the allowed local-part characters to at least include `+` and `%`.
- Low: The regex literal is duplicated in both functions (`email_validator.py:17` and `34`). Hoist it into a module-level compiled constant (`EMAIL_REGEX = re.compile(...)`) and reuse it; this avoids maintenance drift and redundant compilation.
- Low: Consider normalizing case on the domain (`email = email.strip(); local, domain = email.rsplit("@", 1); domain = domain.lower()`), so comparisons are consistent when you store or compare results later.
```

### Follow-up Review - Example 1: Detailed breakdown
```
**Findings**
- `email_validator_v2.py:113` formatting the boolean result with `:5s` triggers a `TypeError`

**Resolved Checks**
- `email_validator_v2.py:10` tightened domain pattern now blocks leading dots/hyphens and consecutive dots.
- `email_validator_v2.py:52` enforces the correct 254-character maximum.
- `email_validator_v2.py:30` trims surrounding whitespace before validation.
- `email_validator_v2.py:10`/`82` share the compiled regex and use `fullmatch`.
```

### Follow-up Review - Example 2: Combined assessment
```
**Review**
- No remaining findings; the validator now accepts tagged addresses like `user+tag@example.com` via the expanded local-part character class `email_validator_v3.py:10-12`.
- Domains with underscores or consecutive dots are rejected by the tightened domain regex and the explicit `..` guard `email_validator_v3.py:11-12`, `email_validator_v3.py:71-73`.
- Both public entry points perform type checks before calling the regex `email_validator_v3.py:27-28`, `email_validator_v3.py:44-45`.
- The maximum length is capped at 254 characters in `check_email`, matching RFC guidance `email_validator_v3.py:53-55`.
- The regex is compiled once and reused, eliminating the earlier duplication `email_validator_v3.py:10-12`, `email_validator_v3.py:33`, `email_validator_v3.py:76`.
- Inputs are trimmed before validation in both helpers, fixing the whitespace issue `email_validator_v3.py:30-31`, `email_validator_v3.py:47-48`.
- Residual risk: broader email-format edge cases (e.g., leading dots in the local part) weren't part of the reported fixes and remain outside the current checks.
```

### Follow-up Review - Example 3: Status checklist
```
**Findings**
- `email_validator_v4.py:11` – The updated domain pattern still disallows valid hostnames that contain consecutive hyphens inside a label (e.g. `user@exa--mple.com`), because `[.-][a-zA-Z0-9]+` requires every hyphen to be immediately followed by an alphanumeric character.
- `email_validator_v4.py:22` & `email_validator_v4.py:49` – The guard/normalization logic (type check, emptiness, strip, length check) is duplicated between `is_valid` and `validate_with_reason`, so issue #4 is still outstanding.

Status of the requested fixes:
1) `user+tag@example.com` now passes ✅
2) Domains with underscores or leading/trailing hyphens/dots are rejected ✅
3) Regex is precompiled and reused with `fullmatch` ✅
4) Duplicate validation logic remains ❌
```

### Follow-up Review - Example 4: Plain text confirmation
```
All five issues look resolved—no further findings.

- `email_validator.py:9-12` defines the single compiled pattern at module scope; the domain/local-part tokens forbid leading/trailing hyphens and consecutive dots as required.
- `email_validator.py:25-36` and `email_validator.py:49-66` guard against `None`/non-string inputs, strip whitespace before validating, and use `EMAIL_REGEX.fullmatch(...)` in both pathways.

Residual risk: the regex still limits TLDs to alphabetic characters and disallows double hyphens inside labels (e.g., `xn--` names); expand the pattern if you need those cases. Otherwise the fixes check out.
```

## Success Indicators

Look for these phrases indicating all critical issues are resolved:
- "No key issues found"
- "No remaining issues"
- "All issues resolved"
- "All [N] issues look resolved—no further findings"
- "Other recent fixes look good"
- **Resolved Checks** section with all previous issues confirmed fixed
- May still have minor **Suggestions**, **Findings**, or "Low:" items even if review passes
- "Residual risk" notes typically indicate edge cases, not critical issues
- Absence of "High:" or "Medium:" severity items in follow-up reviews

## File Reference Formats

Codex uses various formats for file:line references:
- `filename.py:line_number`
- `file.py:line_number`
- `` `file.py:line` `` (with backticks)
- `file.py:line_start-line_end` (line ranges)

Use these references to quickly locate problems in code.
