### Special /Page command with subagents

# Page - Session History Dump with Citations and Memory Management
Like OS paging for processes, this command saves the entire conversation state to disk by extracting it from Claude Code's local storage (`~/.claude/projects/`). After running this command, you can use `/compact` to free up Claude's context memory.

## Usage
```
/project:page [filename_prefix] [output_directory]
```

## Arguments
- `filename_prefix` (optional): Custom prefix for output files. Defaults to "session-dump"
- `output_directory` (optional): Directory to save files. Defaults to `scripts/dev/pages-files`

## Description
This command implements a memory management strategy similar to OS paging:
1. **Page Out (Save to Disk)**:
   - Saves complete conversation state with full citations
   - Creates indexed source references for quick retrieval
   - Preserves all context before memory compaction
2. **Generated Files**:
   - **Full History File** (`{prefix}-{timestamp}-full.md`):
     - Compact summary at top for quick reference
     - Complete conversation transcript with timestamps
     - All file operations with paths and content
     - Web resources with URLs and excerpts
     - Command executions with outputs
     - Full citation index for all sources
   - **Compact Memory File** (`{prefix}-{timestamp}-compact.md`):
     - Executive summary of session
     - Key decisions and outcomes
     - Important code changes made
     - Quick reference links
     - Optimized for future context loading
3. **Memory Management Workflow**:
   - First: Run `/project:page` to save everything to disk
   - Then: Run `/compact` to free up Claude's context memory
   - Result: Fresh context while preserving full history
   - Essential for long development sessions  
**Note**: This prepares for `/compact` by saving everything first. Run `/compact` after this command completes.

## Implementation
**IMPORTANT**: To preserve context window space, execute ALL processing within subagents using the Task tool. Only the final compact summary display and file writes should occur in the main conversation.

### Phase 1: History Extraction from Claude Code Storage
- Checks for Python virtual environment:
  ```bash
  if [ -x ".venv/bin/python" ]; then
      PYTHON_BIN=".venv/bin/python"
  elif [ -x "venv/bin/python" ]; then
      PYTHON_BIN="venv/bin/python"
  else
      echo "Error: Python virtual environment not found."
      exit 1
  fi
  ```
- Ensures extraction script exists at `scripts/dev/pages-files/extract-claude-session.py`:
  ```bash
  if [ ! -f scripts/dev/pages-files/extract-claude-session.py ]; then
      mkdir -p scripts/dev/pages-files
      curl -sSL https://raw.githubusercontent.com/tokenbender/agent-guides/main/scripts/extract-claude-session.py \
          -o scripts/dev/pages-files/extract-claude-session.py
  fi
  ```
- Runs extraction:
  ```bash
  $PYTHON_BIN scripts/dev/pages-files/extract-claude-session.py --latest
  ```

### Phase 2-6: Complete Processing in Subagent
**USE A SUBAGENT (Task tool) TO PERFORM ALL PROCESSING**:
1. Read the extracted Claude session file
2. Parse and analyze content:
   - Count messages, tools used, files accessed
   - Extract key accomplishments and decisions
   - Identify all source files and web resources
   - Note command executions and outputs
3. Generate both compact and full documentation markdown strings
4. Write to `scripts/dev/pages-files/{prefix}-{timestamp}-compact.md` and `scripts/dev/pages-files/{prefix}-{timestamp}-full.md`
5. Return compact summary to main conversation

### Main Conversation
Display only the returned compact summary to the user, then instruct to run `/compact` to free up memory.

### Documentation Templates

**Compact Memory Template**
```markdown
# Session Compact Memory - {timestamp}
## Executive Summary
{2-3 sentence summary of what was accomplished}
## Key Decisions Made
- Decision 1: Reasoning and outcome
- Decision 2: Context and implementation
## Code Changes Summary  
- Feature A: Added functionality X to file Y
- Bug Fix B: Resolved issue Z in component W
## Important Context for Future Sessions
- Project uses framework X with pattern Y
- Key files: config.json, main.py, utils/helpers.py
- Build command: `npm run build`
- Test command: `npm test`
## Quick Reference Links
- [Full History](./{prefix}-{timestamp}-full.md)
- [Key File 1](file:///path/key-file.py)
- [Important Documentation](https://url.com)
## Session Metrics
- Duration: {duration}
- Files touched: {count}
- Major features added: {count}
- Issues resolved: {count}
```

**Full History Template**
```markdown
# Session History - {timestamp}
## Quick Summary (Compact Memory)
### Executive Summary
{2-3 sentence summary of what was accomplished}
### Key Accomplishments
1. **Task 1**: Brief description and outcome
2. **Task 2**: What was done and result
3. **Task 3**: Achievement and impact
### Important Findings
- ✅ Key finding or verification
- 📄 Created file: path/to/file
- 🔧 Fixed issue: description
### Quick Links
- **Main Files**: Links to key files touched
- **Documentation**: Links to docs created/updated
- **References**: External resources used
---
## Full Session Overview
- Start Time: {start}
- Duration: {duration} 
- Total Messages: {count}
- Files Modified: {file_count}
- Web Pages Accessed: {web_count}
- Commands Executed: {cmd_count}
## Conversation Timeline
### Message 1 - User ({timestamp})
{content}
**Sources Referenced:**
- [file.py](file:///path/file.py#L1-L50) - Function implementation
- [Documentation](https://example.com/docs) - API reference
### Message 2 - Assistant ({timestamp})
{content}
**Tools Used:**
- read_file: `/path/to/file.py` (lines 1-50)
- web_search: "claude code best practices" (8 results)
- Bash: `git status` (exit code: 0)
**Files Created/Modified:**
- [new_feature.py](file:///path/new_feature.py) - Created
- [config.json](file:///path/config.json#L15) - Modified line 15
{continue for all messages...}
## Source Index
### Local Files Accessed
1. [file1.py](file:///path/file1.py) - Read 3 times, modified once
2. [config.json](file:///path/config.json) - Modified
### Web Resources  
1. [Claude Code Best Practices](https://anthropic.com/...) - Retrieved Apr 18
2. [GitHub Repository](https://github.com/...) - Searched for examples
### Command Executions
1. `git status` - Check repository state
2. `npm run build` - Build verification  
## Generated Artifacts
- Commands created: 4
- Files created: 2  
- Files modified: 3
```

## Output Format
The command generates two files in `scripts/dev/pages-files`:
1. `{prefix}-{timestamp}-full.md` - Complete history (typically large, with `{timestamp}` as `YYYY-MM-DD_HHMMSS`)
2. `{prefix}-{timestamp}-compact.md` - Executive summary (optimized for context)

## Example Usage
```bash
# Basic usage
/project:page

# Custom prefix
/project:page feature-implementation

# Results saved to scripts/dev/pages-files:
# - feature-implementation-2025-08-13_153022-full.md
# - feature-implementation-2025-08-13_153022-compact.md

# After completion:
 /compact
```
This command is essential for maintaining context across long development sessions and creating comprehensive documentation of AI-assisted development workflows.