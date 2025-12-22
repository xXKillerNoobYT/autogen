# 🧠 Clarification Survey Template

**Purpose:** Gather detailed clarifications when user goal is vague or ambiguous.

**Generated:** {timestamp}
**Plan ID:** {plan_id}
**Vagueness Score:** {vagueness_score}

---

## Question 1: AI Improvement (ALWAYS FIRST)

### How can the AI work better for your workflow?

Please share any friction points, missing features, confusing outputs, or suggestions for improvement. Be specific with examples of what's working and what's not.

**Examples of useful feedback:**
- "The AI assumes I want Python but I mostly use C# - add language detection based on project files"
- "Plans are too high-level with generic steps like 'implement feature'. I need granular steps with specific file names and line numbers"
- "QA questions use too much technical jargon. Simplify language for non-developers or domain experts"
- "Missing integration with Docker/Kubernetes tools I use daily for deployment - suggest adding those MCP servers"
- "Execution stalls when network requests fail - need better retry logic with exponential backoff"
- "I prefer visual diagrams over text descriptions - can you generate mermaid diagrams in plans?"

**Your detailed response (minimum 50 characters, more detail is better):**

```
[ Your feedback here ]
```

**What would make the biggest difference to your productivity?**

```
[ Your answer here ]
```

---

## Question 2: {Contextual Question Based on Vagueness}

{Question text - 2-3 sentences explaining why this matters}

### Option A: {Option Name}

**What this means in practice:**
{50-100 word example showing concrete scenario}

**Example:** {Real-world example with specifics}

**Pros:** {Benefits of this choice}
**Cons:** {Tradeoffs or limitations}

---

### Option B: {Option Name}

**What this means in practice:**
{50-100 word example showing concrete scenario}

**Example:** {Real-world example with specifics}

**Pros:** {Benefits of this choice}
**Cons:** {Tradeoffs or limitations}

---

### Option C: {Option Name}

**What this means in practice:**
{50-100 word example showing concrete scenario}

**Example:** {Real-world example with specifics}

**Pros:** {Benefits of this choice}
**Cons:** {Tradeoffs or limitations}

---

### Option D: Other (Please Specify)

**Guided prompt:** If none of the above options fit your scenario, please describe your specific needs including:
- What success looks like for this aspect
- Any constraints or requirements
- Preferred tools or technologies
- Examples of similar work you've done before

```
[ Your custom response here - be as detailed as needed ]
```

---

**Your choice (A/B/C/D):** [ ]

**Additional context or clarifications:**
```
[ Any extra details that help understand your choice ]
```

---

## Question 3: {Another Contextual Question}

{Repeat same structure as Question 2}

---

## Question 4: {Another Contextual Question}

{Repeat same structure as Question 2}

---

## Question 5: {Another Contextual Question}

{Repeat same structure as Question 2}

---

# 📝 Response Validation

**Before submitting, ensure:**
- [ ] Question 1 (AI Improvement) has detailed feedback (50+ characters)
- [ ] All questions have A/B/C/D selection filled in: [ ]
- [ ] "Other" responses (if selected) have detailed explanations
- [ ] Additional context provided where needed

**Character count check:**
- Q1 response: _____ characters (minimum: 50)
- "Other" responses: _____ characters each (minimum: 50)

---

# 🎯 Next Steps

**After completing this survey:**

1. Save responses to `AI Files/qa_responses.json`
2. Smart Plan agent will:
   - Parse your responses
   - Update goal with specific details
   - Incorporate your AI improvement suggestions into system updates
   - Generate detailed plan based on clarified requirements

**Your responses will:**
- Make the plan more accurate and specific
- Help improve the AI system for future tasks
- Reduce back-and-forth clarifications
- Enable better context gathering

---

**Thank you for taking the time to provide detailed responses!**

The more specific you are, the better the AI can serve your needs.
