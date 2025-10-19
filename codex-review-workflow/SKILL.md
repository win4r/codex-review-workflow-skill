---
name: codex-review-workflow
description: Automated code review workflow using OpenAI Codex CLI. This skill should be used when the user requests code to be reviewed using Codex CLI, or when implementing features that should be validated with automated code review. Implements an iterative fix-and-review cycle until code passes review or reaches iteration limit.
---

# Codex Review Workflow

## Overview

This skill enables automated code review workflows using the OpenAI Codex CLI. Complete coding tasks, automatically submit them for Codex review, fix any identified issues, and re-review until the code passes validation or reaches the maximum iteration limit.

## Workflow Decision Tree

Use this skill when:
- User explicitly requests Codex CLI review (e.g., "Review this with Codex")
- User wants automated code validation after implementation
- Building features that require iterative review and refinement
- Implementing code that must meet specific quality standards

Skip this skill when:
- User only wants manual code review
- No Codex CLI is available in the environment
- Task is purely exploratory or research-based

## Core Workflow

Execute the following steps in order:

### 1. Complete the Coding Task

Implement the user's requested feature or fix using standard best practices. Ensure code is well-structured before submitting for review.

**Important:** Track progress using the TodoWrite tool with these standard tasks:
- Implement the requested feature/fix
- Run initial Codex CLI review
- Fix issues found in review (if any)
- Run final Codex CLI review
- Report final status

### 2. Run Initial Codex CLI Review

**Prerequisites:**
- Codex CLI requires a git repository. If not already in a git repo, run `git init` first
- Alternatively, use `--skip-git-repo-check` flag (not recommended for production code)

Execute the Codex CLI review using the `exec` subcommand with a detailed review prompt:

```bash
# For a specific file - construct a detailed review prompt
codex exec "Review the code in <file_name> for bugs, security issues, best practices, and potential improvements. Provide specific, actionable feedback with line numbers and examples."

# For multiple files - specify them in the prompt
codex exec "Review the files auth.py, user.py, and session.py for bugs, security issues, best practices, and potential improvements. Provide specific feedback for each file."

# With working directory context
codex exec "Review the code in email_validator.py for bugs, security issues, best practices, and potential improvements. Provide specific feedback." -C /path/to/project
```

**Important:**
- Codex CLI does NOT have a `codex review` subcommand - use `codex exec` with a review prompt
- Be specific in your prompt about what to review (bugs, security, performance, etc.)
- Request line numbers and specific examples in the feedback

### 3. Analyze Review Results

Codex CLI returns structured markdown output. **IMPORTANT: The format is highly variable** - Codex uses different section headers at different times. Load `references/codex_output_formats.md` for comprehensive format examples and parsing strategies.

**How to parse the variable output:**

1. **Look for critical issue indicators:**
   - Sections labeled **Bug**, **Security**, **Key Issues**, **Key Findings**
   - Items prefixed with "High:" or "Medium:" severity
   - Any issues with severity indicators (e.g., "critical", "vulnerability")
   - These MUST be fixed

2. **Look for quality improvements:**
   - Sections labeled **Maintainability**, **Usability**, **Best Practices**, **Suggestions**
   - Items prefixed with "Low:" severity
   - These are important but lower priority than bugs/security

3. **Look for confirmation indicators (follow-up reviews):**
   - Sections labeled **Resolved Checks**, **Review**, **Review Findings**
   - Phrases like "No remaining findings", "All issues resolved", "All [N] issues look resolved"
   - Check marks (✅) or confirmation language
   - Statements like "Other recent fixes look good"

4. **Extract actionable information:**
   - File:line references (e.g., `` `file.py:123` ``)
   - Specific problem descriptions
   - Suggested fixes or next steps

**Decision criteria:**

**If the review shows:**
- No **Bug**, **Security**, or **Key Issues** sections → Consider complete if only suggestions remain
- **Resolved Checks** with all previous issues confirmed fixed → Mark workflow complete
- Phrases like "No remaining findings" or "All issues resolved" → Mark workflow complete

**If the review identifies:**
- **Bug**, **Security**, **Key Issues**, or critical problems → Proceed to step 4 to fix them

### 4. Fix Identified Issues

For each issue identified in the Codex review:

1. Locate the problematic code in the relevant file
2. Understand the issue and determine the appropriate fix
3. Apply the fix using the Edit tool
4. Document what was changed and why

**Best practices:**
- Fix all issues in a single iteration before re-reviewing
- Prioritize critical errors over warnings
- Explain each fix clearly to the user
- Preserve code functionality while addressing issues

### 5. Run Follow-up Codex CLI Review

After applying fixes, run the Codex CLI review again with a prompt that asks about the specific issues:

```bash
# Verify specific fixes were applied correctly
codex exec "Review the updated <file_name> code. Check if the previous issues have been fixed: 1) issue description, 2) issue description, etc. Report any remaining issues or confirm the code is now correct."
```

This targeted prompt helps Codex:
- Focus on whether the specific issues were resolved
- Identify any new problems introduced by the fixes
- Provide clear pass/fail confirmation

Analyze the new results:
- **If Codex confirms all issues are fixed:** Mark the workflow as complete and report success
- **If issues remain or new issues were introduced:** Determine whether to iterate again (see iteration limits below)

### 6. Handle Iteration Limits

**Maximum iterations:** 2 review cycles (initial + 1 re-review)

**Why limit iterations:**
- Prevents infinite loops
- Some issues may require human judgment
- Complex problems might need architectural changes

**When maximum iterations are reached:**
1. Mark the current task status in TodoWrite
2. Summarize what was fixed and what issues remain
3. Show the user the final Codex CLI output
4. Ask the user whether to:
   - Continue with manual fixes
   - Adjust the approach
   - Accept the current state

## Examples

### Example 1: Simple Fix Workflow

```
User: "Add a function to calculate fibonacci numbers and review it with Codex"

Claude:
1. Creates the fibonacci function
2. Checks for git repo, runs git init if needed
3. Runs: codex exec "Review fibonacci.py for bugs, edge cases, and best practices. Provide specific feedback."
4. Codex output:
   - **Bug** `fibonacci.py:5`: Missing input validation for negative numbers
   - **Maintainability** `fibonacci.py:10`: Consider adding type hints
5. Adds input validation and type hints
6. Runs: codex exec "Review the updated fibonacci.py. Check if input validation for negative numbers has been added. Report any remaining issues."
7. Codex output:
   **Resolved Checks**
   - `fibonacci.py:5` Input validation is now present
   - `fibonacci.py:10` Type hints added
8. Reports completion
```

### Example 2: Multiple Iterations

```
User: "Implement user authentication and run it through Codex review"

Claude:
1. Implements authentication system across multiple files
2. Runs: codex exec "Review auth.py, user.py, and session.py for security vulnerabilities, bugs, and best practices."
3. Codex output:
   - **Security** `auth.py:23`: Password stored without hashing
   - **Security** `session.py:45`: Session tokens don't expire
   - **Bug** `user.py:67`: SQL injection vulnerability in login query
   - **Security** `auth.py:89`: Missing CSRF protection
   - **Bug** `user.py:12`: No input validation on username field
4. Fixes all 5 issues
5. Runs: codex exec "Review the updated auth.py, user.py, and session.py. Check if these issues are fixed: 1) password hashing, 2) session expiration, 3) SQL injection prevention, 4) CSRF protection, 5) input validation."
6. Codex output:
   **Resolved Checks**
   - `auth.py:23` Password hashing implemented with bcrypt
   - `session.py:45` Session expiration added (30min timeout)
   - `user.py:67` Parameterized queries prevent SQL injection
   - `auth.py:89` CSRF tokens implemented
   - `user.py:12` Input validation added

   **Findings**
   - `auth.py:15` Consider using more descriptive variable names
   - `user.py:34` Function could benefit from docstring
7. Reached iteration limit (2 reviews)
8. Reports status: "All critical security issues fixed. Minor style suggestions remain."
9. Asks user whether to continue with style fixes
```

### Example 3: First-Pass Success

```
User: "Write a helper function to format dates, then review with Codex"

Claude:
1. Creates date formatting function with proper error handling
2. Runs: codex exec "Review utils/date_formatter.py for bugs, edge cases, and best practices."
3. Codex output:
   No bugs or critical issues found. The code properly handles:
   - None values with appropriate error messages
   - Invalid date format strings
   - Timezone conversions

   **Suggestions**
   - `utils/date_formatter.py:45` Could add more detailed error messages for debugging
4. Reviews suggestion, decides it's optional for current use case
5. Marks task complete
```

## Integration with Codex CLI

### Standard Commands

The Codex CLI uses the `exec` subcommand for code review (NOT `codex review`):

```bash
# Basic review command
codex exec "Review <file_name> for bugs, security issues, best practices, and potential improvements. Provide specific, actionable feedback with line numbers and examples."

# Review with working directory context
codex exec "Review <file_name>..." -C /path/to/project

# Review with specific model (optional)
codex exec "Review <file_name>..." -m gpt-5-codex

# Review with custom configuration
codex exec "Review <file_name>..." -c model="o3"
```

### Git Repository Requirement

Codex CLI requires the working directory to be a git repository:

```bash
# Check if in git repo
git status

# If not, initialize one
git init

# Alternative: skip git check (not recommended)
codex exec "..." --skip-git-repo-check
```

### Parsing Codex Output

For comprehensive format examples and real-world output variations, load `references/codex_output_formats.md` when encountering unfamiliar output patterns.

### Handling Edge Cases

**Codex CLI not found:**
- Check if Codex CLI is installed: `which codex` or `codex --version`
- Inform user that Codex CLI is not available
- Offer to complete the task without automated review

**Git repository error:**
- Error message: "Not inside a trusted directory and --skip-git-repo-check was not specified"
- Solution: Run `git init` to initialize a git repository
- Alternative: Add `--skip-git-repo-check` flag (not recommended)

**Codex CLI errors:**
- If Codex CLI itself errors (not code review failures), show the error
- Common errors:
  - `unexpected argument` - Check command syntax, ensure using `codex exec` not `codex review`
  - Authentication errors - User may need to run `codex login`
- Attempt once more with corrected parameters
- If persistent, ask user for guidance

**Ambiguous results:**
- If unsure whether Codex output indicates pass/fail, err on side of caution
- Look for both "Key Issues" and "Suggestions" sections - presence of "Key Issues" means problems exist
- Show output to user and ask for clarification

**Long-running reviews:**
- Codex may take 30-120 seconds for complex reviews
- Use appropriate timeout values (120000ms = 2 minutes recommended)

## Best Practices

1. **Always use TodoWrite** to track workflow steps for user visibility
2. **Show Codex output** to user at each review stage
3. **Explain fixes clearly** - avoid silent fixes
4. **Respect iteration limits** - avoid infinite loops
5. **Preserve functionality** - address issues without breaking features
6. **Ask when uncertain** - consult user when Codex feedback is ambiguous

## Customization

Customize this workflow by:
- Adjusting iteration limits (default: 2 reviews)
- Specifying custom Codex CLI commands
- Providing a configuration file for Codex rules
- Defining which files to include/exclude from review
- Setting severity thresholds (e.g., only fix errors, not warnings)
