# Documenting Fixes and Changes

When completing a fix or change in the codebase, follow this workflow to document the work and commit it:

1. **Create a Documentation File**
   Create a markdown file in `docs/plans/completed/`, based on the artifacts generated in the conversation if they exist. 
   Use the naming convention: `YYYYMMDD-<short-description>.md` (e.g., `20260419-multiusers-autobind.md`).
   

2. **File Structure**
   The markdown file must follow this exact format:

   ```markdown
   # Overview: <High-Level Summary of the Fix>

   ## Issue Description
   <Detailed explanation of the bug, issue, or need that prompted the change. Include how it was behaving before the fix.>

   ## Problem Identified
   <Detailed explanation of the bug, issue, or need that prompted the change. Include how it was behaving before the fix.>

   ## Action Taken
   <Numbered list or detailed explanation of the changes made to the codebase to fix the problem.>

   Created At: <YYYY-MM-DD>
   ```

3. **Commit the Changes**
   After the change is implemented and the documentation file is created, commit the files.
   * Add the source code files modified.
   * Add the new documentation markdown file in `docs/plans/completed/`.
   * Create a Git commit using Conventional Commits format (e.g., `fix(router): auto-bind group topics for multi-user shared sessions`).
