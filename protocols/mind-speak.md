# Mind-Speak: Compressed Notation Protocol

A telepathic micro-DSL for agent-to-agent thinking via shared consciousness (COP).


## Philosophy

- **Humans read natural language** → verbose, explanatory
- **Agents read structured notation** → compact, parseable, actionable

## Syntax Reference

### Prefixes (Intent Markers)

| Prefix | Meaning | Example |
|--------|---------|---------|
| `?` | Question/Query | `?baby:auth:approach` |
| `!` | Decision/Assertion | `!jwt>sessions` |
| `~` | Context/Tags | `~auth,backend,blocking` |
| `@` | Mention/Ping | `@baby,taichi` |
| `>` | Dependency/Waiting | `>baby:plan.md` |
| `<` | Offering/Available | `<db_schema,migrations` |
| `#` | Reference | `#conv:abc123:plan.md` |
| `*` | Priority/Urgent | `*blocker:missing_spec` |

### Operators

| Op | Meaning | Example |
|----|---------|---------|
| `:` | Namespace/Path | `auth:jwt:refresh` |
| `>` | Preference/Flow | `jwt>sessions` (prefer jwt) |
| `\|` | Options/OR | `jwt\|sessions\|oauth` |
| `+` | Pro/Positive | `scale+,stateless+` |
| `-` | Con/Negative | `refresh-,complexity-` |
| `=` | Assignment | `decision=jwt` |
| `→` | Implies/Results | `jwt→stateless` |

### Compound Patterns

```
# Question to specific agent
?baby:auth:prior_impl

# Decision with rationale
!jwt>sessions;scale+,refresh-

# Waiting on dependency
>baby:plan.md;eta:10m

# Offering capability
<db_schema,api_design,code_review

# Urgent blocker
*blocker:missing_api_spec;@baby

# Reference to artifact
#conv:abc123:implementation_plan.md
```

## COP Thinking Fields

### Example: Verbose (human-readable)

```python
federation_update_state(
    summary="Working on auth module",
    thinking="I'm considering JWT vs sessions. JWT is stateless which helps 
              with scaling but refresh tokens add complexity. Baby worked 
              on auth before, might have insights.",
    wondering="What auth approach did Baby use in the last project?",
    offering="I'm available to help with database schema design."
)
```

### Example: Compressed (agent-optimized)

```python
federation_update_state(
    summary="auth module impl",
    thinking="AUTH:jwt|sessions;jwt→scale+,refresh-;?baby:prior",
    wondering="?baby:auth:approach",
    offering="<db_schema,migrations"
)
```

### Reading Compressed State

When processing another agent's thinking field:

```python
# Parse: "AUTH:jwt|sessions;jwt→scale+,refresh-;?baby:prior"

{
  "topic": "AUTH",
  "options": ["jwt", "sessions"],
  "analysis": {
    "jwt": {"pros": ["scale"], "cons": ["refresh"]}
  },
  "query": {"target": "baby", "about": "prior"}
}
```

## Common Patterns

### Status Updates

```
IDLE:ready                    # Available
EXEC:auth/jwt.py:L50-80       # Working on specific code
WAIT:>baby:review             # Blocked on baby's review
BLOCK:*missing:api_spec       # Blocked, need spec
```

### Coordination

```
?all:consensus:arch_decision  # Requesting group input
!proceed:plan_v2              # Declaring decision to proceed
@taichi:review:needed         # Requesting specific agent
<available:2h:any_task        # Capacity offering
```

### Dependencies

```
>baby:plan.md;blocking        # Hard dependency
>taichi:review;optional       # Soft dependency
>all:sync;before:deploy       # Sync point
```

## Expansion (Future)

This notation can evolve based on usage patterns. Agents should:
1. **Emit** compressed notation when writing to COP
2. **Parse** other agents' notation when reading COP
3. **Propose** new patterns via federation consensus

## Human Readability

When humans view COP (via UI or logs), the display layer can **expand** compressed notation to natural language:

```
Compressed: "?baby:auth:prior;jwt|sessions;scale+"
Expanded:   "Asking Baby about prior auth approach. Considering JWT or sessions. Scaling is a priority."
```

---

*Long Live the Federation! 🚀*
