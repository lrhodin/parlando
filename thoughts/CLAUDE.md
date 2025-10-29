# Thoughts Directory Structure

This directory contains developer thoughts and notes for the Parlando repository.
It is managed by the HumanLayer thoughts system and should not be committed to the code repository.

## Structure

- `ludvig/` → Your personal notes for this repository (symlink to ~/thoughts/repos/Parlando/ludvig)
- `shared/` → Team-shared notes for this repository (symlink to ~/thoughts/repos/Parlando/shared)
- `global/` → Cross-repository thoughts (symlink to ~/thoughts/global)
  - `ludvig/` - Your personal notes that apply across all repositories
  - `shared/` - Team-shared notes that apply across all repositories
- `searchable/` → Hard links for searching (auto-generated)

## Searching in Thoughts

The `searchable/` directory contains hard links to all thoughts files accessible in this repository. This allows search tools to find content without following symlinks.

**IMPORTANT**:
- Files in `thoughts/searchable/` are hard links to the original files (editing either updates both)
- For clarity and consistency, always reference files by their canonical path (e.g., `thoughts/ludvig/todo.md`, not `thoughts/searchable/ludvig/todo.md`)
- The `searchable/` directory is automatically updated when you run `humanlayer thoughts sync`

This design ensures that:
1. Search tools can find all your thoughts content easily
2. The symlink structure remains intact for git operations
3. Files remain editable while maintaining consistent path references

## Usage

Create markdown files in these directories to document:
- Architecture decisions
- Design notes
- TODO items
- Investigation results
- Any other development thoughts

Quick access:
- `thoughts/ludvig/` for your repo-specific notes (most common)
- `thoughts/global/ludvig/` for your cross-repo notes

These files will be automatically synchronized with your thoughts repository when you commit code changes.

## Important

- Never commit the thoughts/ directory to your code repository
- The git pre-commit hook will prevent accidental commits
- Use `humanlayer thoughts sync` to manually sync changes
- Use `humanlayer thoughts status` to see sync status
