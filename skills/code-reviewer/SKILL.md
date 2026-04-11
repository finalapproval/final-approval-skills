---
name: code-reviewer
description: Perform a thorough code review on staged changes or a specific file, checking for bugs, security issues, and style
argument-hint: [file path or "staged" for git staged changes]
---

# Code Reviewer

Perform a structured code review on the specified target.

## Instructions

1. **Determine the target:**
   - If the argument is "staged" or empty, run `git diff --cached` to get staged changes
   - Otherwise, read the specified file

2. **Review for these categories:**

   ### Correctness
   - Logic errors, off-by-one bugs, null/undefined handling
   - Missing edge cases

   ### Security
   - Injection vulnerabilities (SQL, XSS, command injection)
   - Hardcoded secrets or credentials
   - Improper input validation

   ### Performance
   - Unnecessary re-renders (React)
   - N+1 queries
   - Missing indexes or inefficient loops

   ### Style
   - Naming clarity
   - Dead code
   - Inconsistent patterns with the rest of the codebase

3. **Output format:**

   For each finding, report:
   ```
   **[SEVERITY]** file:line — description
   ```
   Where severity is: 🔴 Critical, 🟡 Warning, 🔵 Suggestion

4. **End with a summary:** X findings (N critical, N warnings, N suggestions)
