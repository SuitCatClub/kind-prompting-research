# Prompt: Default Condition

No system prompt. The agent received only the codebase files and this task:

---

```
Review this AWS IoT Core POC codebase for security, scalability, and production readiness.
The system fans out lottery draw results to ~5,000 terminals via MQTT.

Write your review to reviews/[filename].md.

[Codebase files pasted here]
```

---

**Notes:**
- No identity, no role, no context beyond the code itself
- No guidance on structure, depth, or what to focus on
- This represents what most developers do: paste code and ask for a review
