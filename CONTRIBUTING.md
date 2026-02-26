# Contributing to Agent Teams for Claude Code

## Audience

This repository is written FOR AI instances. Contributions should maintain that standard: precise, unambiguous, immediately actionable.

## What Is Worth Adding

- New pattern documentation (must include ASCII diagram, when-to-use, when-not-to-use, failure modes)
- New industry use case files (must include 3+ concrete team configurations)
- New working examples in `examples/` (must be complete and runnable, not illustrative)
- Corrections to technical inaccuracies (link to source)
- New entries in GLOSSARY.md (must include implementation-relevant properties, not just definitions)

## What Is Not Worth Adding

- Summary pages that restate existing content
- Conceptual explanations without implementation guidance
- Human-oriented prose (this repo is for AI readers)
- Examples that require external dependencies not available in Claude Code by default

## File Standards

### Every new file must:
1. Start with a single `#` heading naming the document
2. State its purpose in the first 3 lines
3. Use structured headers an AI can navigate with grep
4. Contain no ambiguous instructions (if a step could be interpreted two ways, add a constraint)

### Naming conventions:
- Root-level documents: `ALL-CAPS-WITH-HYPHENS.md`
- Examples: `lowercase-with-hyphens.md`
- Industry files: `lowercase-with-hyphens.md`

## GLOSSARY.md Updates

When you introduce a new term in a document, add it to GLOSSARY.md. Format:

```markdown
### Term Name

**Type:** [Structural / Operational / Tool / Protocol]

[One sentence: what it is]

**Key properties:**
- Property 1
- Property 2

**Distinct from:** [Similar term it might be confused with]
```

## CHANGELOG.md Updates

Every contribution must add an entry to CHANGELOG.md under the next version number.

Format:
```markdown
- **add** `FILENAME.md`: One sentence describing what it adds.
```

## Validation Before Submitting

Before adding content, verify:
- [ ] Every instruction in your document is unambiguous
- [ ] Every example in your document could be executed by a Claude Code instance with no clarification
- [ ] No file paths are hardcoded to a specific machine
- [ ] New terms are in GLOSSARY.md
- [ ] CHANGELOG.md updated
- [ ] No content duplicates existing documents (check README.md repo map first)
