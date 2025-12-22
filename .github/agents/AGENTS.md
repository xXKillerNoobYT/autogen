# AGENTS.md - Multi-Agent Orchestration System Documentation

## Overview

This documentation describes the **5-Agent AI Orchestration System** built for GitHub Copilot's Agent HQ, enabling intelligent planning, execution, review, and cloud coordination through file-state-driven orchestration.

---

## 🏗️ System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      USER GOAL / REQUEST                          │
└────────────────────────────┬─────────────────────────────────────┘
                             ↓
        ╔════════════════════════════════════════╗
        ║     FULL AUTO (Optional Orchestrator)   ║
        ║  Chains Plan → Execute → Review Loop    ║
        ╚════════════════┬═══════════════════════╝
                         ↓
        ┌────────────────────────────────────────┐
        │      SMART PLAN (Planning Agent)        │
        │  • Vagueness detection (0-1 score)     │
        │  • QA survey generation (if needed)    │
        │  • Context gathering                   │
        │  • Plan generation with steps          │
        │  • Confidence scoring                  │
        │  ↓ Outputs: plan_output.json           │
        └────────────────────────────────────────┘
                         ↓
        ┌────────────────────────────────────────┐
        │    SMART EXECUTE (Execution Agent)      │
        │  • Loads plan_output.json               │
        │  • Runs steps sequentially              │
        │  • Invokes MCP tools                    │
        │  • Edits Plan/Review/FullAuto prompts  │
        │  • Records observations to MCP memory  │
        │  ↓ Outputs: execution_state.json       │
        └────────────────────────────────────────┘
                         ↓
        ┌────────────────────────────────────────┐
        │    SMART REVIEW (Review Agent)          │
        │  • Analyzes execution results           │
        │  • Root-cause analysis on stalls       │
        │  • Updates Smart Execute prompt        │
        │  • Proposes replanning (if needed)     │
        │  • Tracks 10-cycle evaluation          │
        │  ↓ Outputs: replanning_context.json    │
        └────────────────────────────────────────┘
                         ↓
        ┌────────────────────────────────────────┐
        │  SMART PREP CLOUD (Cloud Handoff Agent) │
        │  • Environment readiness analysis      │
        │  • GitHub Issue generation             │
        │  • Cloud Confidence scoring (5 signals)│
        │  • TODO breadcrumbs in-file           │
        │  ↓ Outputs: cloud_prep_output.json    │
        └────────────────────────────────────────┘
```

---

## 📋 Agent Reference

### 1. Smart Plan
**File:** `.github/agents/Smart Plan.agent.md`

**Purpose:** Analyzes user goals, detects vagueness, asks clarifying questions, and generates detailed execution plans.

**YAML Metadata:**
```yaml
name: Smart Plan
version: 1.0.0
description: Intelligent planning agent with vagueness detection (0-1 scoring), QA survey generation, context gathering, plan generation, and handoff management
argument-hint: Describe your goal or problem to plan
tools: [vscode, execute, read, edit, search, web, mcp_docker/*, agent, memory, todo]
handoffs:
  - label: Start Execution
    agent: Smart Execute
    prompt: Begin executing the plan with all context and observations
  - label: Smart Prep Cloud
    agent: Smart Prep Cloud
    prompt: Prepare a cloud handoff - generate GitHub Issue, add TODO breadcrumbs, validate environment, compute Cloud Confidence
```

**Key Phases:**
1. **Phase 1: Vagueness Detection** - Analyzes goal using 6 signals, produces 0-1 score
2. **Phase 2: QA Survey** - If vagueness > 0.7, generates survey questions
3. **Phase 3: Context Gathering** - Uses file_search, semantic_search, MCP queries
4. **Phase 4: Plan Generation** - Creates ordered steps with tool specifications
5. **Phase 5: Confidence Scoring** - Rates plan quality across 5 signals

**Outputs:**
- `AI Files/plan_output.json` - Executable plan with steps, considerations, confidence
- Next-agent recommendation with confidence percentage

**Handoff Triggers:**
- "Start Execution" → Smart Execute (when plan ready)
- "Smart Prep Cloud" → Smart Prep Cloud (when cloud execution needed)

---

### 2. Smart Execute
**File:** `.github/agents/Smart Execute.agent.md`

**Purpose:** Loads execution plans and runs steps sequentially, with capability to edit other agent prompts based on findings.

**YAML Metadata:**
```yaml
name: Smart Execute
version: 1.0.0
description: Execution agent that loads plans, executes steps using tools, edits Plan/Review/FullAuto prompts, records observations
argument-hint: Execute plan with full context
tools: [vscode, execute, read, edit, search, web, mcp_docker/*, pylance-mcp-server/*, agent, memory, todo, etc.]
handoffs:
  - label: Complete Execution Review
    agent: Smart Review
    prompt: Analyze execution results and identify stalls or improvements
```

**Key Phases:**
1. **Phase 1: Plan Loading & Validation** - Reads plan_output.json, validates structure
2. **Phase 2: Step-by-Step Execution** - For each step: invoke tools, record observations
3. **Phase 3: Prompt Updating** - Edit Plan, Review, FullAuto (NOT itself)
4. **Phase 4: MCP Memory Recording** - Log observations with timestamps
5. **Phase 5: Handoff** - Hand off to Smart Review

**Outputs:**
- `AI Files/execution_state.json` - Tracks step completion, stalls, failures
- `AI Files/prompt_edit_log.jsonl` - Records all prompt modifications
- `AI Files/pending_updates.json` - Medium-confidence prompt updates for review
- `AI Files/old_versions/` - Backup copies of edited agent files

**Key Rule:**
- **CANNOT edit itself** (Smart Execute.agent.md)
- **CAN edit:** Smart Plan.agent.md, Smart Review.agent.md, Full Auto.agent.md

---

### 3. Smart Review
**File:** `.github/agents/Smart Review.agent.md`

**Purpose:** Analyzes execution results, performs root-cause analysis, updates Smart Execute prompt, and proposes replanning.

**YAML Metadata:**
```yaml
name: Smart Review
version: 1.0.0
description: Review agent that analyzes execution stalls, performs root-cause analysis, updates Smart Execute prompt, proposes replanning
argument-hint: Review execution results and propose improvements
tools: [vscode, execute, read, edit, search, web, agent, memory, todo]
handoffs:
  - label: Replan with Insights
    agent: Smart Plan
    prompt: Create new plan incorporating learnings and root-cause analysis
```

**Key Phases:**
1. **Phase 1: Execution Analysis** - Reads execution_state.json, checks for stalls
2. **Phase 2: Root-Cause Analysis** - Asks 5 diagnostic questions on stalls
3. **Phase 3: Smart Execute Updates** - Edits Execute prompt (only agent it modifies)
4. **Phase 4: Replanning Proposal** - Generates replanning_context.json (if needed)
5. **Phase 5: 10-Cycle Evaluation** - Every 10 cycles, creates comprehensive evaluation

**Outputs:**
- `AI Files/replanning_context.json` - Root causes and recommendations
- `AI Files/10_cycle_evaluation_{timestamp}.md` - Analysis every 10 cycles
- `AI Files/prompt_edit_log.jsonl` - Appends Smart Execute edits

**Key Rule:**
- **ONLY edits:** Smart Execute.agent.md
- **CANNOT edit:** Smart Plan, FullAuto, or itself

---

### 4. Smart Prep Cloud
**File:** `.github/agents/Smart Prep Cloud.agent.md`

**Purpose:** Prepares comprehensive cloud handoff artifacts for autonomous GitHub Actions execution.

**YAML Metadata:**
```yaml
name: Smart Prep Cloud
version: 1.0.0
description: Prepares cloud handoff artifacts including issue generation, environment validation, Cloud Confidence scoring
tools: [file_search, read_file, semantic_search, mcp_mcp_docker_issue_write, mcp_mcp_docker_add_observations]
handoffs:
  - name: Cloud Agent
    description: Hand off via @cloud mention or /delegate command
    confidence_threshold: 0.75
```

**Key Phases:**
1. **Phase 1: Environment Readiness** - Checks workflow, secrets, MCP, runner, artifacts
2. **Phase 2: GitHub Issue Generation** - Creates execution issue with exact commands
3. **Phase 3: TODO Breadcrumbs** - Adds in-file TODOs for cloud-side navigation
4. **Phase 4: Cloud Confidence Scoring** - Computes percentage (5 signals × 20%)
5. **Phase 5: Handoff Recommendation** - Determines mode based on confidence

**Cloud Confidence Scoring Formula:**

$$C = \sum_{i=1}^{5} w_i \cdot s_i$$

Where: $w_i = 0.20$ (equal weight), and signals are:
1. **Workflow Readiness:** Does `.github/workflows/*.yml` exist and reference cloud agents?
2. **Secrets Availability:** Are required secrets (API keys, tokens) configured?
3. **MCP Allowlist:** Is Docker MCP Gateway allowlisted for cloud environment?
4. **Runner Compatibility:** Can workflow run on `runs-on: ubuntu-latest`?
5. **Artifact Completeness:** Are all required input files present (plan, config, etc.)?

**Confidence Thresholds:**
- **≥75%**: PROCEED with cloud execution
- **50-75%**: Use cloud WITH LOCAL FALLBACK
- **<50%**: PREFER LOCAL execution
- **<25%**: DO NOT USE cloud (critical issues)

**Outputs:**
- `AI Files/cloud_prep_output.json` - Environment analysis and confidence score
- GitHub Issue (if proceeding) - Commands and acceptance criteria
- MCP observations - Cloud readiness status

---

### 5. Full Auto
**File:** `.github/agents/Full Auto.agent.md`

**Purpose:** Orchestrator that autonomously chains Plan → Execute → Review until task completion.

**YAML Metadata:**
```yaml
name: Full Auto
version: 1.0.0
description: Orchestrator agent that chains Smart Plan → Execute → Review until completion, validates version sync, manages workflow autonomously
argument-hint: Fully automate task from planning to completion
tools: [vscode, execute, read, edit, search, web, agent, mcp_docker/*, memory, todo, etc.]
handoffs:
  - label: Start Planning
    agent: Smart Plan
    prompt: Begin planning with user goal and context
```

**Key Phases:**
1. **Phase 1: Initialization** - Load all 3 agent files, validate version sync
2. **Phase 2: Orchestration Loop** - Chain Plan → Execute → Review (max 10 cycles)
3. **Phase 3: Result Processing** - Analyze outcomes, decide continue/stop
4. **Phase 4: Cycle Tracking** - Log cycles, trigger 10-cycle evaluation
5. **Phase 5: Completion** - Declare success or escalate to user

**Loop Logic:**
```
FOR each cycle (max 10):
  1. RUN Smart Plan (or skip if replanning_context.json exists)
  2. RUN Smart Execute with plan
  3. RUN Smart Review with execution_state.json
  4. IF replanning_context.json exists:
       CONTINUE loop with Smart Plan
     ELSE:
       BREAK (task complete)
  5. IF cycle_count % 10 == 0:
       GENERATE 10_cycle_evaluation_*.md
       PROPOSE improvements
```

**Outputs:**
- Coordinates outputs from all 3 agents
- Manages cycle tracking in state files
- Final status: success/failure/user_stop

---

## 🎯 Confidence Scoring System

### Smart Plan Confidence Scoring

Smart Plan produces confidence scores for its generated plans using a formula:

$$\text{Confidence} = \sum_{i=1}^{5} w_i \cdot s_i$$

**5 Scoring Signals** (each 0-1):

| Signal | Description | Scoring Rubric |
|--------|-------------|-----------------|
| **Context Quality** | Gathered sufficient context? | 0 = minimal, 1 = comprehensive |
| **Step Clarity** | Are steps unambiguous? | 0 = vague, 1 = crystal clear |
| **Tool Coverage** | Right tools specified? | 0 = wrong, 1 = optimal match |
| **Dependency Order** | Correct sequence? | 0 = incorrect, 1 = perfect order |
| **Completeness** | Any gaps? | 0 = missing pieces, 1 = complete |

**Weight:** $w_i = 0.20$ (equal distribution)

**Score Interpretation:**
- **0.8-1.0**: High confidence → Direct to Smart Execute
- **0.6-0.8**: Medium confidence → Smart Execute with fallback
- **0.4-0.6**: Low confidence → Recommend Smart Review before Execute
- **<0.4**: Very low confidence → Suggest replanning

**Example Calculation:**
```
Signal scores: [0.9, 0.85, 0.8, 0.9, 0.88]
Confidence = (0.2 × 0.9) + (0.2 × 0.85) + (0.2 × 0.8) + (0.2 × 0.9) + (0.2 × 0.88)
           = 0.18 + 0.17 + 0.16 + 0.18 + 0.176
           = 0.866 (86.6% - HIGH confidence → Smart Execute recommended)
```

---

## 🔄 Execution Workflow Examples

### Example 1: Local Development Mode (Synchronous)

**User Goal:** "Create comprehensive AGENTS.md documentation"

**File-State-Driven Flow:**

```
Step 1: User triggers Smart Plan (via IDE handoff/chat)
        ↓
        Smart Plan READS: .github/agents/ files, memory observations
        Smart Plan WRITES: plan_output.json ← (File system state update)
        Smart Plan OUTPUTS: "Ready for execution" + next-agent recommendation
        ↓

Step 2: User (or Full Auto) reads plan_output.json (file system checks)
        TRIGGERS Smart Execute with confidence info from plan
        ↓
        Smart Execute READS: plan_output.json
        Smart Execute EXECUTES: 6 steps (create files, edit prompts, etc.)
        Smart Execute WRITES: execution_state.json ← (File system state update)
        Smart Execute OUTPUTS: "6 steps completed" + next-agent recommendation
        ↓

Step 3: User (or Full Auto) reads execution_state.json
        TRIGGERS Smart Review
        ↓
        Smart Review READS: execution_state.json, agent files
        Smart Review WRITES: replanning_context.json (if needed)
        Smart Review OUTPUTS: "Analysis complete - task successful"
        ↓

Step 4: User (or Full Auto) checks if replanning needed
        NO replanning_context.json → Task complete
        TASK COMPLETE ✓
```

**State Files Created:**
- ✅ `plan_output.json` (from Plan)
- ✅ `execution_state.json` (from Execute)
- ✅ `.github/agents/AGENTS.md` (artifact created)

---

### Example 2: Background Execution Mode (Asynchronous via GitHub Actions)

**User Goal:** "Set up CI/CD pipeline"

**File-State-Driven Flow:**

```
Local IDE (VS Code)                    GitHub Actions Runner
───────────────────                    ─────────────────────

1. User writes goal in chat
   Smart Plan runs locally
   → plan_output.json written to repo
                                      
2. Workflow trigger (on git push)
   GitHub Actions starts
   ↓
   Smart Prep Cloud validates
   → cloud_prep_output.json written
   → GitHub Issue created with issue #123
   ↓
   Smart Execute runs in runner
   (reads plan from git clone)
   → execution_state.json written
   → Artifacts uploaded
   ↓
   Smart Review analyzes
   → Comment on issue #123
   
3. User checks issue #123
   Reads execution_state.json artifact
   ← Knows execution status
```

**State Files:**
- ✅ `plan_output.json` (from Plan)
- ✅ `cloud_prep_output.json` (from Prep Cloud)
- ✅ `execution_state.json` (from Execute in runner)
- ✅ GitHub Issue with comments/artifacts

---

### Example 3: Cloud Autonomous Mode (Copilot Cloud Agent)

**User Goal:** "Deploy to production"

**File-State-Driven Flow:**

```
VS Code IDE                            GitHub Copilot Cloud
───────────────                        ────────────────────

1. User requests deployment
   Smart Plan generates plan
   → plan_output.json written

2. Smart Prep Cloud validates
   Cloud Confidence = 82% (≥75%) ✓
   → cloud_prep_output.json written
   → GitHub Issue #456 created
   → TODO breadcrumbs added in files
   ↓
   
3. Smart Prep Cloud handoffs to Cloud Agent
   (via @cloud mention in issue or /delegate)
   
4. Cloud Agent receives:
   - plan_output.json (plan details)
   - cloud_prep_output.json (environment validation)
   - GitHub Issue #456 (commands, acceptance criteria)
   - TODO breadcrumbs (navigation hints)
   ↓
   Cloud Agent runs independently:
   - No IDE context needed
   - Executes in isolated runner
   - Writes execution_state.json
   - Posts results as issue comment
   
5. Execution completes
   User reviews issue #456
   Reads execution_state.json comment
   → Deployment status visible
```

**Artifacts in Cloud Handoff:**
- 📄 `plan_output.json` (what to do)
- 📄 `cloud_prep_output.json` (environment status)
- 📝 GitHub Issue with exact commands
- 🔖 TODO breadcrumbs in relevant files

---

## 📊 File-State-Driven Orchestration Principle

### Core Design: No Runtime State, Only File I/O

Unlike traditional orchestration that passes state objects between agents via function calls, this system uses **file-based state coordination**:

**❌ Anti-Pattern (Traditional):**
```
Plan() → returns plan_object
         ↓
Execute(plan_object) → runs with in-memory data
                       ↓
Review(execution_result) → checks runtime object
```
*Problem:* Breaks in background/cloud execution (no IDE process)

**✅ Pattern (File-State-Driven):**
```
Plan() → WRITES plan_output.json to disk
         ↓
         [File system is single source of truth]
         ↓
Execute() → READS plan_output.json
            WRITES execution_state.json
         ↓
         [File system updated with new state]
         ↓
Review() → READS execution_state.json
           WRITES replanning_context.json
           
[Can run locally, background, or cloud - any process can read files]
```

### Key State Files

| File | Owner | Reader | Purpose |
|------|-------|--------|---------|
| `plan_output.json` | Smart Plan | Smart Execute, Full Auto, Cloud | What to execute (steps, tools, dependencies) |
| `execution_state.json` | Smart Execute | Smart Review, Full Auto, Cloud | Step completion status, failures, stalls |
| `replanning_context.json` | Smart Review | Smart Plan, Full Auto | Root causes, recommendations for next cycle |
| `cloud_prep_output.json` | Smart Prep Cloud | Cloud Agent, user | Environment readiness, Cloud Confidence score |
| `prompt_edit_log.jsonl` | Smart Execute | Smart Review, audit | All prompt modifications with backups |
| `10_cycle_evaluation_*.md` | Smart Review | User, Full Auto | Pattern analysis every 10 cycles |

### How Prompting Keeps Files Updated

**The contract between agents:**

1. **Smart Plan** reads files → runs analysis → **WRITES plan_output.json**
   - Prompting ensures: "Save your plan to `AI Files/plan_output.json` as valid JSON"
   
2. **Smart Execute** reads plan → runs steps → **WRITES execution_state.json**
   - Prompting ensures: "After each step, update `AI Files/execution_state.json` with completion status"
   
3. **Smart Review** reads execution results → analyzes → **WRITES replanning_context.json**
   - Prompting ensures: "If replanning needed, save root causes to `AI Files/replanning_context.json`"

**Prompting ensures file consistency through explicit instructions:**
```
In your prompts:
✓ "Save {output} to {path} as {format}"
✓ "Update {file} by appending {structure}"
✓ "After {action}, write {file} with {schema}"
✓ "Keep {file} as single source of truth for {data}"
```

### How File Reading Triggers Next Steps

**The orchestration trigger mechanism:**

1. **Full Auto polls for file state changes:**
   ```
   FOR each cycle:
       IF plan_output.json exists:
           → Smart Plan completed, advance to Execute
       ELSE IF plan_output.json NOT exist:
           → Trigger Smart Plan
           
       IF execution_state.json exists:
           → Smart Execute completed, advance to Review
       ELSE IF plan_output.json exists AND execution_state.json NOT:
           → Trigger Smart Execute
           
       IF replanning_context.json exists:
           → Smart Review completed, cycle again with new plan
       ELSE IF execution_state.json exists AND replanning_context.json NOT:
           → Trigger Smart Review
   ```

2. **User manually triggers by reading files:**
   ```
   User reads plan_output.json in IDE
   → Visible next-agent recommendation in file
   → User decides to run Smart Execute
   → OR uses Full Auto which automates this
   ```

3. **CI/CD triggers via file changes:**
   ```
   GitHub Actions workflow watches for file modifications
   IF execution_state.json updated:
       → Trigger post-execution review job
       → Generate comment on PR/Issue
   ```

### Benefits of File-State-Driven Design

| Benefit | Explanation |
|---------|-------------|
| **Portable** | Works in IDE, GitHub Actions, Cloud agents, any process |
| **Inspectable** | Files visible in repo, diffs show exactly what changed |
| **Debuggable** | Can read state files to understand where orchestration stuck |
| **Asynchronous** | Agents don't need to be running simultaneously |
| **Version Controlled** | State changes tracked in git for audit trail |
| **No Shared Memory** | Each agent is stateless except for file I/O |
| **Cloud Compatible** | Cloud agents get files via git clone, write via push |

---

## 🛠️ Troubleshooting Guide

### Issue 1: Plan Generation Stalls (Vagueness Detection Loop)

**Symptom:** Smart Plan runs but never outputs plan_output.json. Stuck on vagueness detection.

**Root Causes:**
1. Vagueness score never drops below 0.7 after QA
2. Context gathering returns empty results
3. MCP memory queries fail

**Solutions:**

**If Vagueness Stays High:**
```
1. Check qa_responses.json - were answers provided?
   IF NO: Ask user to answer QA survey questions
   
2. Review vagueness scoring logic in plan output
   IF goal fundamentally ambiguous: Ask user for clarification
   
3. Manually lower vagueness threshold temporarily
   EDIT Smart Plan.agent.md:
   Change: "IF vagueness_score > 0.7: generate_qa_survey()"
   To: "IF vagueness_score > 0.85: generate_qa_survey()"
```

**If Context Gathering Fails:**
```
1. Check file_search patterns - are paths correct?
   VALIDATE: .github/, python/, dotnet/ paths exist
   
2. Check semantic_search - is workspace indexed?
   RUN: VS Code command "Semantic Search: Index Workspace"
   
3. Check MCP memory - is Docker MCP Gateway running?
   RUN: mcp-find in terminal to verify MCP servers
```

**If MCP Queries Fail:**
```
1. Verify MCP allowlist includes search tools
   EDIT: Smart Plan.agent.md tools list
   ADD: mcp_mcp_docker_search_nodes
   
2. Check memory has observations
   RUN: Search MCP memory manually
   IF empty: Prior executions didn't record observations
```

**Recovery Action:**
```
Reset plan generation by:
1. DELETE AI Files/plan_output.json
2. DELETE AI Files/qa_responses.json
3. RUN Smart Plan again with same goal
```

---

### Issue 2: Execution Stalls on Specific Step

**Symptom:** Smart Execute completes step 1, 2, 3 but Step 4 never finishes. execution_state.json shows step_3 completed but step_4 has timestamp > 5 minutes.

**Root Causes:**
1. Tool invocation failed silently
2. File operation locked or permission denied
3. MCP tool unavailable
4. Step dependencies not met

**Solutions:**

**Check execution_state.json for error details:**
```json
{
  "step": 4,
  "action": "create_file /some/path/file.md",
  "status": "in-progress",
  "duration_ms": 300000,
  "error": "Permission denied: /some/path/file.md",
  "tool_used": "create_file"
}
```

**If Tool Unavailable:**
```
1. Check available MCP servers
   RUN: mcp-find in terminal
   
2. If tool missing, activate it
   RUN: mcp-add {server_name}
   
3. Update Smart Execute tools list
   EDIT: Smart Execute.agent.md → tools section
   ADD: {new_mcp_tool}/*
```

**If File Operation Failed:**
```
1. Check file path is absolute
   VALIDATE: Path starts with / (Linux) or C: (Windows)
   
2. Check parent directory exists
   CREATE: Parent directories with create_directory tool
   
3. Check write permissions
   VERIFY: User has write access to target directory
```

**If Dependencies Not Met:**
```
1. Review step dependencies in plan_output.json
   EXAMPLE: Step 5 requires Step 3 file, but Step 3 failed
   
2. Re-run Smart Review
   Smart Review will detect dependency issue
   WILL: Propose replanning with corrected order
```

**Recovery Action:**
```
Manual recovery for stalled step:
1. Diagnose issue using execution_state.json details
2. EITHER:
   a) Fix root cause directly (permissions, missing file, etc.)
   b) Skip step and continue: Manually edit execution_state.json
      Change step_4.status = "skipped"
      Smart Execute will resume from step_5
3. Re-run Smart Execute (will skip completed steps)
```

---

### Issue 3: Prompt Editing Creates Conflicts

**Symptom:** After Smart Execute runs, one of the edited agent files (Smart Plan, Review, FullAuto) is broken. Syntax errors, missing sections, version mismatch.

**Root Causes:**
1. Old string not found - section header changed
2. New string broke YAML formatting
3. Multiple edits overlapped
4. Backup restoration needed

**Solutions:**

**If Section Not Found:**
```
1. Check prompt_edit_log.jsonl for recent edits
   VIEW: Last edit to target agent
   
2. Verify old_string in backup file
   READ: AI Files/old_versions/{agent}_{timestamp}.md
   COMPARE: Is header exact match?
   
3. If header changed, Smart Execute cannot find it
   SOLUTION: Don't let Smart Plan self-update itself!
   (Smart Plan can be updated by Smart Review only)
```

**If YAML Formatting Broken:**
```
1. Check execution_state.json - did edit complete?
   IF status = "failed": Edit did not complete
   
2. Restore from backup
   COPY: AI Files/old_versions/{agent}_{timestamp}.md
   TO: .github/agents/{agent}.agent.md
   
3. Review edit proposal
   READ: prompt_edit_log.jsonl last failed edit
   WHY: What was the change trying to accomplish?
   
4. Manual fix recommended
   Edit the agent file manually to apply change
```

**If Version Mismatch:**
```
Full Auto detected version != 1.0.0
1. All 4 agent files must have version: 1.0.0
2. After prompt edits, version should remain 1.0.0
3. If version changed, revert to 1.0.0

Verify with command:
grep "version: " .github/agents/*.agent.md
# All should show version: 1.0.0
```

**Recovery Action:**
```
To recover from prompt edit failure:

1. Identify failed agent
   READ: prompt_edit_log.jsonl last line
   
2. Restore original
   FIND: AI Files/old_versions/{agent}_{oldest}.md
   COPY: → .github/agents/{agent}.agent.md
   
3. Verify syntax
   CHECK: No red squiggles in editor
   VALIDATE: YAML frontmatter valid
   
4. Log recovery
   APPEND to prompt_edit_log.jsonl:
   {
     "action": "recovery",
     "recovered_agent": "{agent}",
     "backup_used": "AI Files/old_versions/{agent}_{timestamp}.md",
     "reason": "Edit failed - restored from backup",
     "timestamp": "2025-..."
   }
```

---

### Issue 4: Cloud Confidence Too Low (<50%)

**Symptom:** Smart Prep Cloud generates cloud_prep_output.json but Cloud Confidence = 35% (below 50% threshold). Recommends local execution even though you wanted cloud.

**Root Causes:**
1. GitHub workflow file missing or misconfigured
2. Required secrets not found
3. MCP allowlist not configured
4. Runner incompatible
5. Required artifacts missing

**Solutions:**

**Identify which signal is failing:**
```json
{
  "signals": {
    "workflow_readiness": 1.0,      // ✓ Workflow file exists
    "secrets_availability": 0.0,     // ✗ NO secrets found!
    "mcp_allowlist": 0.5,            // ~ MCP partially configured
    "runner_compatibility": 1.0,     // ✓ Can run on ubuntu-latest
    "artifact_completeness": 0.8     // ✓ Most files present
  },
  "confidence": (1.0 + 0.0 + 0.5 + 1.0 + 0.8) / 5 = 0.66... wait, should be 0.35
```

**For Each Failed Signal:**

**Workflow Readiness (should be 1.0):**
```
IF 0.0: .github/workflows/ missing or no cloud-agent trigger
SOLUTION:
  1. Create .github/workflows/cloud-execute.yml
  2. Add trigger: on: [workflow_dispatch, push]
  3. Add step that runs Smart Execute or Cloud Agent
  4. Commit and push

IF 0.5: Workflow exists but no cloud agent reference
SOLUTION:
  1. Edit workflow to include cloud agent step
  2. Add: @cloud /run-plan or /delegate
```

**Secrets Availability (should be 1.0):**
```
IF 0.0: No secrets configured in repo
SOLUTION:
  1. GitHub Settings → Secrets and Variables → Actions
  2. ADD required secrets: API keys, tokens, credentials
  3. Update workflow to read secrets
  4. Common secrets: OPENAI_API_KEY, GITHUB_TOKEN

IF 0.5: Some secrets present, some missing
SOLUTION:
  1. Identify which secrets cloud plan needs
  2. Review cloud_prep_output.json → required_secrets list
  3. Add missing secrets to GitHub Settings
```

**MCP Allowlist (should be 1.0):**
```
IF 0.0: Docker MCP Gateway not allowlisted for cloud
SOLUTION:
  1. Copilot Cloud Settings → MCP Allowlist
  2. ADD: mcp_mcp_docker
  3. ALLOW all tools: mcp_mcp_docker*
  4. Test: Try delegating to cloud agent

IF 0.5: MCP partially configured
SOLUTION:
  1. Review which mcp_mcp_docker/* tools allowed
  2. Ensure these are enabled:
     - mcp_mcp_docker_get_overview
     - mcp_mcp_docker_search_tasks
     - mcp_mcp_docker_create_task
     - mcp_mcp_docker_search_nodes
```

**Runner Compatibility (should be 1.0):**
```
IF 0.0 or 0.5: Runner environment incompatible
SOLUTION:
  1. Plan likely requires specific OS or runtime
  2. Edit workflow: runs-on: [ubuntu-latest, windows-latest, macos-latest]
  3. Ensure runner has required tools installed
  4. EXAMPLE: Python plans need python action in workflow

COMMON ISSUES:
  - Python 3.x not available
  - .NET SDK not installed
  - Required CLI tools missing
  - Docker not available
```

**Artifact Completeness (should be 1.0):**
```
IF < 1.0: Required input files missing
SOLUTION:
  1. Check required_artifacts in cloud_prep_output.json
  2. Ensure these files exist in repo:
     - plan_output.json (from Smart Plan)
     - Any config files referenced in plan
     - Source files that plan depends on
  3. Commit missing files to repo
  4. Re-run Smart Prep Cloud
```

**Recovery Action:**
```
To improve Cloud Confidence:

1. Identify failing signals from cloud_prep_output.json
2. Fix each signal issue (see solutions above)
3. Commit changes to repo
4. DELETE cloud_prep_output.json
5. RUN Smart Prep Cloud again
6. Check new confidence score
7. IF ≥75%: PROCEED with cloud execution
   IF 50-75%: Use WITH LOCAL FALLBACK
   IF <50%: Stick with LOCAL execution
```

---

## 🚀 Quick Start

1. **Run Smart Plan** with your goal
   - Plan analyzes goal, detects vagueness
   - Outputs `plan_output.json`

2. **Trigger Smart Execute**
   - Executes plan steps
   - Outputs `execution_state.json`

3. **Check Smart Review**
   - Analyzes execution
   - Proposes improvements or declares complete

4. **Optional: Cloud Prep**
   - Smart Prep Cloud validates cloud readiness
   - Generates Cloud Confidence score
   - Creates GitHub Issue for cloud delegation

5. **Or: Full Auto**
   - Chains all agents automatically
   - Runs up to 10 cycles
   - Completes end-to-end

---

## 📚 References

- **Agent Files:** `.github/agents/*.agent.md`
- **State Files:** `.github/agents/AI Files/*.json`
- **Visual Diagrams:** `.github/agents/AI Files/ARCHITECTURE.md`
- **System Constraints:** `/memories/system_constraints.md`

---

**Last Updated:** December 21, 2025  
**Version:** 1.0.0  
**Maintained By:** Smart Execute Agent
