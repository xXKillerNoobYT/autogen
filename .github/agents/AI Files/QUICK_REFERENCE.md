# 🚀 Quick Reference Card - 4-Agent AI System

**One-page guide for using the self-updating AI system**

---

## ⚡ Quick Start

**Run a task:**
```
Load Full Auto.agent.md and give it your goal
```

**That's it!** Full Auto handles everything:
- Loads other 3 agents dynamically
- Chains them in the right order
- Manages the workflow loop
- Archives results when done

---

## 🔄 How It Works (Simplified)

```
Your Goal
    ↓
┌─────────────────────────────────────┐
│ Smart Plan                          │
│ • Detects vagueness                 │
│ • Generates QA if needed            │
│ • Gathers context                   │
│ • Creates step-by-step plan         │
│ • Proposes prompt updates           │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Smart Execute                       │
│ • Executes plan steps               │
│ • Uses MCP tools                    │
│ • Edits other agent prompts         │
│ • Records observations              │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ Smart Review                        │
│ • Analyzes execution                │
│ • Detects stalls/failures           │
│ • Edits Execute agent               │
│ • Manages cycle counter             │
│ • Triggers 10-cycle evaluations     │
└──────────────┬──────────────────────┘
               ↓
    ┌──────────┴──────────┐
    │ Complete or Replan? │
    └──────────┬──────────┘
               │
        Complete → Done ✓
        Replan   → Loop back to Plan
```

---

## 📂 File Quick Reference

| File | Purpose | Who Uses |
|------|---------|----------|
| `cycle_metadata.json` | Cycle counter, evaluation tracking | Review, Full Auto |
| `prompt_edit_log.jsonl` | Immutable edit audit trail | Execute, Review |
| `plan_output.json` | Plan with steps and update proposals | Plan → Execute |
| `execution_state.json` | Execution progress tracking | Execute → Review |
| `qa_survey_template.md` | Template for clarification surveys | Plan |
| `schemas/*.json` | Validation schemas for all files | All agents |

---

## 🧠 Where Data Lives

**MCP Memory** (primary):
- ✅ Observations and learnings
- ✅ Semantic search history
- ✅ Cross-session patterns
- ✅ Temporary insights

**Local Files** (secondary):
- ✅ Audit trails (prompt_edit_log)
- ✅ State persistence (cycle_metadata)
- ✅ Compliance records
- ✅ Structured data

**Rule of thumb:** Hot data → MCP memory, Cold data → Local files

---

## 🔍 Common Scenarios

### "My goal is vague"

Smart Plan will:
1. Detect vagueness (score >0.7)
2. Generate QA survey
3. **Always ask first:** "How can AI work better?"
4. Ask 4-5 contextual questions with A/B/C/Other options
5. Incorporate your answers into detailed plan

**Example vague goals:**
- "Make the system better"
- "Fix issues"
- "Improve performance"

### "Execution stalled"

Smart Review will:
1. Detect if step takes >2 minutes
2. Perform root-cause analysis
3. Propose fix to Smart Execute prompt
4. Trigger replanning if needed

### "I want to see system improvements"

Check these files:
- `AI Files/prompt_edit_log.jsonl` - All edits made
- `AI Files/10_cycle_evaluation_*.md` - Every 10 cycles
- MCP memory - Search for "systemic improvement"

### "Version mismatch error"

Full Auto detects this at startup:
1. Reads version from each agent's YAML frontmatter
2. Compares versions
3. Alerts if mismatch
4. **Fix:** Update outdated agent prompts to match

---

## 🛠️ Troubleshooting

| Problem | Solution |
|---------|----------|
| Agent not finding files | Check workspace path, use file_search tool first |
| MCP tools failing | Verify Docker MCP gateway running, use mcp-find to discover |
| Prompt edit failed | Check prompt_edit_log.jsonl error_message field |
| QA survey not generating | Vagueness score <0.7 (goal clear enough) |
| Cycle not incrementing | Smart Review didn't complete successfully |
| Version validation fails | Update agent YAML frontmatter versions to match |

---

## 📊 Monitoring Health

**Good signs:**
- ✅ Cycle count incrementing regularly
- ✅ Plans completing within expected time
- ✅ Prompt edits with high confidence (>0.7)
- ✅ 10-cycle evaluations showing improvements

**Warning signs:**
- ⚠️ Multiple replans for same goal
- ⚠️ Stalls >5 minutes per step
- ⚠️ Same error recurring across cycles
- ⚠️ Low confidence edits (<0.5)

**Critical issues:**
- 🚨 Infinite replan loop
- 🚨 Version mismatches blocking execution
- 🚨 Prompt_edit_log.jsonl corruption
- 🚨 MCP memory connection failures

---

## 🎯 Best Practices

**Do:**
- ✅ Provide detailed goals (reduces QA surveys)
- ✅ Answer QA surveys thoroughly (saves cycles)
- ✅ Review 10-cycle evaluations (systemic insights)
- ✅ Keep agent versions synchronized
- ✅ Monitor prompt_edit_log for patterns

**Don't:**
- ❌ Edit prompt_edit_log.jsonl manually (append-only!)
- ❌ Skip QA surveys (leads to vague plans)
- ❌ Ignore version mismatch warnings
- ❌ Delete cycle_metadata.json (loses history)
- ❌ Run agents individually (use Full Auto orchestrator)

---

## 🔐 Security & Compliance

**Audit Trail:**
- `prompt_edit_log.jsonl` = immutable record of all changes
- Never modify existing lines
- Only append new entries
- Preserves full edit history

**Data Retention:**
- MCP memory: Ephemeral (can be cleared)
- Local files: Permanent (version controlled)
- Archive old sessions to `AI Files/archive/`

**Version Control:**
- Commit agent prompt changes
- Commit schema updates
- Don't commit execution_state.json (temporary)
- Commit cycle_metadata.json (historical record)

---

## 📖 Quick Command Reference

**Check cycle status:**
```json
cat "AI Files/cycle_metadata.json" | jq '.cycle_count, .next_evaluation_due'
```

**View recent edits:**
```bash
tail -n 10 "AI Files/prompt_edit_log.jsonl" | jq
```

**Find observations in MCP:**
```
Use: mcp_mcp_docker_search_nodes with query="root cause analysis"
```

**Validate JSON files:**
```bash
# Against schemas
jsonschema -i cycle_metadata.json schemas/cycle_metadata.schema.json
```

---

## 🆘 Emergency Procedures

**System stuck in loop:**
1. Check cycle_metadata.json - if cycle_count not changing, Review agent failed
2. Check execution_state.json - find failed step
3. Review prompt_edit_log.jsonl - recent edits may have introduced bug
4. Restore from `AI Files/old_versions/` if needed

**Prompt corruption:**
1. Stop execution immediately
2. Restore from `AI Files/old_versions/[agent].bak`
3. Review prompt_edit_log.jsonl to find bad edit
4. Add validation to prevent recurrence

**Data loss:**
1. MCP memory: Check for reconnection issues
2. Local files: Restore from version control
3. Check `AI Files/archive/` for backups

---

## 📞 Getting Help

1. **Check this card first** (you are here)
2. **Read `AI Files/README.md`** for detailed explanations
3. **Review agent prompts** for workflow details
4. **Search MCP memory** for similar issues encountered before
5. **Check 10-cycle evaluations** for systemic patterns

---

**Version:** 1.0.0  
**Last Updated:** 2025-01-10  
**Maintained By:** 4-Agent Self-Updating System

---

💡 **Tip:** Keep this card open while working with the system!
