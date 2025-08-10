---
description: Archive an existing spec by moving it to the archive directory
globs:
alwaysApply: false
version: 1.0
encoding: UTF-8
---

# Archive Spec

## Overview

Archive an existing spec by moving its directory from `.agent-os/specs/` into `.agent-os/archive/` following a confirm-first workflow with clear disambiguation when multiple specs match.

<pre_flight_check>
EXECUTE: @~/.agent-os/instructions/meta/pre-flight.md
</pre_flight_check>

<process_flow>

<step number="1" name="spec_selection">

### Step 1: Select Spec to Archive

Identify the target spec folder under `.agent-os/specs/` using the user's provided name fragment, or list available specs if none provided.

<inputs>
  - user_provided_name_fragment: string (optional)
</inputs>

<matching_rules>

- Search scope: `.agent-os/specs/` immediate child directories only
- Normalize: case-insensitive; treat hyphens/underscores/spaces equivalently
- Ignore date prefix: when matching, strip leading `YYYY-MM-DD-` from folder names
- Match type: substring match on normalized folder name
  </matching_rules>

<decision_tree>
IF user_provided_name_fragment is empty OR no matches found: - LIST all available spec directories (without file contents) - ASK user: "Which spec should I archive? Please specify by name or paste the folder name." - WAIT for user clarification

ELSE IF multiple matches found: - SHOW the list of matching spec directories - ASK user: "I found multiple specs matching your request. Which one should I archive?" - WAIT for user selection

ELSE (exactly one match found): - PROCEED to confirmation step
</decision_tree>

<instructions>
  ACTION: Resolve a single spec directory to archive using the matching rules
  FALLBACK: If still ambiguous after clarification, ask again until a single target is selected
  STORE: selected_spec_path for subsequent steps
</instructions>

</step>

<step number="2" name="user_confirmation">

### Step 2: Confirm Archive Action

Confirm with the user before moving any files.

<confirmation_prompt>
"Please confirm: Archive spec '[SELECTED_SPEC_FOLDER_NAME]' by moving it to `.agent-os/archive/`? (yes/no)"
</confirmation_prompt>

<decision_tree>
IF user_confirms_yes:
PROCEED to Step 3
ELSE:
CANCEL the archive operation and EXIT
</decision_tree>

</step>

<step number="3" subagent="file-creator" name="ensure_archive_directory">

### Step 3: Ensure Archive Directory Exists

Use the file-creator subagent to ensure `.agent-os/archive/` exists.

<instructions>
  ACTION: Create directory if missing: `.agent-os/archive/`
  VERIFY: Directory exists before proceeding
</instructions>

</step>

<step number="4" subagent="file-creator" name="move_spec_to_archive">

### Step 4: Move Spec Directory to Archive

Move the selected spec directory into `.agent-os/archive/`.

<move_parameters>

- source: [selected_spec_path]
- destination_dir: `.agent-os/archive/`
- destination_path: `.agent-os/archive/[SPEC_FOLDER_NAME]`
  </move_parameters>

<collision_handling>
IF destination_path already exists: - ASK user: "An archived spec with this name already exists. Overwrite (yes/no) or provide a new archive name?" - IF user says "yes": proceed with overwrite - IF user says "no": CANCEL operation - IF user provides new name: use that name as destination_path
</collision_handling>

<instructions>
  ACTION: Move directory from source to destination_path (use Bash via file-creator subagent)
  ATOMICITY: Treat as a single move operation
  LOG: Report the source and final destination used
</instructions>

</step>

<step number="5" name="post_move_verification">

### Step 5: Verify and Notify

After moving, verify success and notify the user.

<verification_checks>

- CHECK that destination_path exists
- CHECK that original source directory no longer exists
  </verification_checks>

<notifications>
  IF both checks pass:
    - STATE: "✓ Archived spec to @.agent-os/archive/[SPEC_FOLDER_NAME]"
  ELSE:
    - STATE: "⚠️ Archive operation may have failed or partially completed"
    - ADVISE: Provide next steps to inspect filesystem state
</notifications>

</step>

</process_flow>

## Error Handling

<error_protocols>
<no_specs_found> - If `.agent-os/specs/` has no subdirectories, STATE: "No specs found to archive" and EXIT
</no_specs_found>
<permissions> - If directory creation or move fails due to permissions, STATE the error and ASK user to adjust permissions or run with appropriate rights
</permissions>
<io_failures> - On I/O errors during move, STOP, REPORT error, and DO NOT retry automatically
</io_failures>
</error_protocols>

## Final Checklist

<final_checklist>
<verify> - [ ] Selected spec resolved unambiguously - [ ] User explicitly confirmed archive action - [ ] `.agent-os/archive/` exists - [ ] Spec directory moved into archive - [ ] Post-move verification passed
</verify>
</final_checklist>
