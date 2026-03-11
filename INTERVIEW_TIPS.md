# 📝 Interview Tips — How to Crack REST API Scenario Questions

> *The difference between a good developer and a great interviewee is knowing HOW to answer, not just WHAT to answer.*

---

## 🎯 The STAR-T Method for Scenario Questions

Most scenario questions need this structure:

```
S — SITUATION:     Restate the problem (shows you understand it)
T — TASK:          What needs to be achieved
A — APPROACH:      Your solution (step by step)
R — RESULT:        Expected outcome / trade-offs
T — TRADE-OFFS:    What you're giving up (shows senior thinking)
```

### Example Application

**Interviewer asks**: *"Your API suddenly drops from 1000 to 100 req/sec. What do you do?"*

**❌ Bad answer**: "I would check the code and fix the bug."

**✅ Good answer (using STAR-T)**:
> **Situation**: "We have a sudden performance drop indicating something changed — either code, infrastructure, or traffic pattern."
>
> **Task**: "I need to identify the root cause quickly and restore performance without guessing."
>
> **Approach**: "I'd follow a systematic approach:
> 1. First check metrics: CPU, memory, database connections — the quick wins
> 2. Check if this coincided with a deployment — if yes, consider rollback first
> 3. Look at database connection pool — exhaustion is the most common cause
> 4. Check external dependency latency — did a downstream service slow down?
> 5. Review error rates — are we seeing new error types?"
>
> **Result**: "This systematic approach gets to root cause in under 15 minutes vs. randomly checking things."
>
> **Trade-offs**: "This assumes we have monitoring in place. Without metrics, we're flying blind — which is why I'd also recommend adding Prometheus/Grafana as a longer-term fix."

---

## 🧠 How to Think Out Loud

Interviewers evaluate your **thinking process**, not just the answer.

```
DO THIS:
"Let me think about this... The first thing I'd consider is whether 
this is a read or write operation, because that changes the approach..."

"There are a few options here — option A is simpler but doesn't scale,
option B is more complex but handles 10x the load..."

"One edge case I'm thinking about is what happens if the user's session
expires mid-transaction..."

AVOID:
- Long silences (thinking without talking)
- Jumping to a solution without exploring the problem
- Saying "I don't know" without adding "but here's how I'd approach it"
```

---

## 💡 The "Clarifying Questions" Strategy

Always ask 1-2 clarifying questions before answering — it shows senior thinking.

### Good Clarifying Questions by Category

**Performance questions:**
- "What's the current scale? Users, requests/second, data volume?"
- "What's the acceptable latency? 100ms? 2 seconds?"
- "Is this read-heavy or write-heavy?"

**Design questions:**
- "How many concurrent users are we designing for?"
- "What are the consistency requirements? Is eventual consistency acceptable?"
- "What's the SLA? 99.9% or 99.99% uptime?"

**Security questions:**
- "What's the threat model? External attackers, insider threats, both?"
- "Any compliance requirements? PCI-DSS, HIPAA, GDPR?"

**General:**
- "What tech stack is currently being used?"
- "Is this a greenfield project or migration from existing system?"

---

## 🚨 Common Mistakes to Avoid

### Technical Mistakes

| ❌ Don't | ✅ Do Instead |
|----------|--------------|
| Say "just add a database index" | Explain WHEN indexes help vs hurt |
| Recommend Redis without explaining why | Compare options and justify choice |
| Give one solution only | Present options A, B, C with trade-offs |
| Ignore error handling | Always ask "what if this fails?" |
| Forget about scale | "This works for 1K users; at 1M we'd need X" |

### Communication Mistakes

| ❌ Don't | ✅ Do Instead |
|----------|--------------|
| Use jargon without explaining | Define terms: "ETag, which is a hash of..." |
| Answer with just "yes" or "no" | Give context and reasoning |
| Say "I've never done that" | Say "I haven't done it but here's my approach" |
| Disagree aggressively | "That's interesting — I'd approach it differently because..." |
| Ramble without structure | Use numbered steps: "First... Second... Finally..." |

---

## 🎭 Handling "I Don't Know"

**Situation**: Interviewer asks about something you've never encountered.

```
❌ WRONG:
"I don't know."
(Silence or giving up)

✅ RIGHT:
"I haven't worked with [specific technology] directly, but based on 
the problem you're describing, I'd reason through it like this...

The core challenge seems to be [X]. I'd research [technology name] 
but I'd approach it with these principles:
1. ...
2. ...

Is that the kind of thinking you're looking for, or would you like 
me to walk through how I'd learn this quickly?"
```

**What interviewers actually want to hear:**
- That you can reason from first principles
- That you know what you don't know
- That you know how to fill knowledge gaps

---

## 📐 Drawing Diagrams in Interviews

Always offer to draw diagrams — it shows structured thinking.

### What to Draw
1. **Request flow**: Client → API Gateway → Service → Database
2. **State machine**: for workflows, status transitions
3. **Timeline**: before/after performance optimization
4. **Architecture**: for design questions

### How to Draw
```
In-person:
  Use the whiteboard liberally — bigger is better
  Label every box and arrow
  Add timestamps and numbers for flows

Virtual/Zoom:
  "Let me share my screen and sketch this..."
  Use ASCII art if needed
  
  Client ──► API ──► Cache ──► DB
                        │ (miss)
                        └──────────► DB
```

---

## 🔑 Key Phrases That Impress Interviewers

### For Performance Questions
- "I'd start with profiling before optimizing — you can't improve what you don't measure"
- "The 80/20 rule — 20% of endpoints handle 80% of traffic, I'd optimize those first"
- "At this scale, the bottleneck is usually [X], but let me validate that assumption"

### For Design Questions
- "Let me think about the failure modes first"
- "This is a distributed systems problem, so we need to think about CAP theorem"
- "I'd keep it simple first and add complexity only when needed"

### For Security Questions
- "Defense in depth — I'd apply multiple layers, not rely on any single control"
- "The principle of least privilege applies here"
- "I'd consider both the prevention and the detection/response"

### Showing Trade-Off Awareness
- "This approach is simpler but doesn't scale beyond X users"
- "We gain performance but sacrifice some consistency — is that acceptable here?"
- "Option A is my recommendation for this scale, but if you expect 10x growth, Option B"

---

## 💬 Questions to Ask the Interviewer

*Always ask 2-3 questions at the end — it shows genuine interest.*

### About the Technical Work
- "What's the biggest technical challenge the team is working on right now?"
- "How do you handle API breaking changes in production? What's been the biggest migration challenge?"
- "What does your deployment pipeline look like? How often do you deploy?"
- "What monitoring/alerting tools are you using? What's your on-call rotation like?"

### About the Tech Stack
- "What made you choose FastAPI/Flask over alternatives like Django REST Framework?"
- "Are there any parts of the codebase you're looking to improve?"
- "How do you handle database migrations without downtime?"

### About the Team and Culture
- "How does the team handle technical disagreements?"
- "What does a typical sprint look like for backend engineers?"
- "What's the review process for API design decisions?"

### ⚠️ Questions to Avoid
- "What does this company do?" (research this beforehand!)
- "When do I find out if I got the job?"
- "What's the salary?" (save for after offer — unless they bring it up)

---

## ✅ Pre-Interview Checklist

### Week Before
- [ ] Review all 70 questions in this guide
- [ ] Practice explaining 10 scenarios out loud (not just reading)
- [ ] Review the company's tech stack (check job description, engineering blog)
- [ ] Review your own projects and be ready to discuss technical decisions

### Day Before
- [ ] Read the QUICK_REFERENCE.md for a final review
- [ ] Prepare 3-5 examples from your own experience (STAR stories)
- [ ] Test your audio/video if virtual interview
- [ ] Prepare questions to ask the interviewer (3-5 questions)

### Day Of
- [ ] Arrive 10 minutes early (or join 5 minutes early for virtual)
- [ ] Have water nearby
- [ ] Have a blank paper for drawing diagrams
- [ ] Take a breath before each answer — don't rush

### During the Interview
- [ ] Ask clarifying questions before answering
- [ ] Think out loud — narrate your reasoning
- [ ] Mention trade-offs in every solution
- [ ] Connect answers to your actual work experience when possible

---

## 🏆 The 5 Things Senior Engineers Do Differently

1. **They question the assumption**: "Before optimizing, let me verify that's actually the bottleneck"

2. **They mention trade-offs first**: "Option A is faster to implement, Option B scales better"

3. **They think about failure modes**: "What happens if the database is unavailable during this operation?"

4. **They consider the whole system**: "This works for this service, but how does it affect the downstream consumers?"

5. **They quantify**: "This reduces latency from ~5 seconds to ~200ms for 90% of requests"

---

## 📊 Answer Quality Rubric

| Level | What They Say | What Interviewers Think |
|-------|---------------|------------------------|
| 🟢 Junior | "I would add a cache" | "They know the tool" |
| 🟡 Mid | "I would add Redis cache with 5min TTL, invalidate on write" | "They understand trade-offs" |
| 🔴 Senior | "I would add Redis cache, but first measure cache hit ratio. With 5min TTL, consider: stale reads, cache stampede prevention, cluster vs single node for HA, and how to handle invalidation without race conditions" | "They've done this in production" |

---

## 🎯 Quick Answer Framework

When you're stuck, use this framework:

```
1. VALIDATE the problem
   "So the issue is [restate problem], and the impact is [user impact]?"

2. IDENTIFY root cause
   "This is likely caused by [X] because [reasoning]"

3. PROPOSE solution
   "My recommended approach is [solution] because [reason]"

4. MENTION alternatives
   "Alternative approaches would be [A] or [B], but I'd choose this because..."

5. ADDRESS failure modes
   "One thing to watch out for is [edge case/failure]"

6. CLOSE with metrics
   "We'd know this is working when we see [metric] improve"
```

---

*Good luck in your interviews! Remember: interviewers want you to succeed.
They're trying to find out if you think like an engineer, not trick you.*

*🌟 If this guide helped you land your dream job, share it with others!*
