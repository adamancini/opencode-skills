---
name: knowledge-ingest
description: Use when the user says "learn this", "teach claude this", "remember this article", "add this to my knowledge base", "distill this", or provides a URL to ingest into the vault and curated reference library.
version: 0.1.0
metadata:
  author: adamancini
  repository: https://github.com/adamancini/opencode-skills
  tags: knowledge,ingest
  globs: ""
  alwaysApply: "false"
---


# Knowledge Ingestion from URL

Fully automated 9-step workflow that ingests a URL into both the personal Obsidian vault and the devops-toolkit curated knowledge-base reference library.

**Execute ALL steps without asking for confirmation.** Only pause if domain or vault placement is genuinely ambiguous.

## Constants

| Name | Path |
|------|------|
| Vault | `~/notes/` |
| Knowledge base | `~/.config/opencode/plugins/marketplaces/devops-toolkit/plugins/devops-toolkit/skills/knowledge-base/` |
| KB SKILL.md | `<knowledge-base>/SKILL.md` |
| KB references | `<knowledge-base>/reference/` |
| devops-toolkit repo | `~/.config/opencode/plugins/marketplaces/devops-toolkit/` |

## Domain Inference

- Work signals: Kubernetes, DevOps, Replicated, Helm, infra, platform, KOTS, cloud → `work/`
- Personal signals: homelab, hobbies, RV, personal finance, health → `personal/`
- If ambiguous, ask once: "Should this go in work/ or personal/?"

## Steps

**1. Fetch and analyze** — Use WebFetch on the URL. Extract: title, all technical details, procedures, commands, key concepts, trade-offs, links to official docs.

**2. Classify content**
- Domain: `work` or `personal`
- Vault subdirectory: e.g., `work/kubernetes/`, `personal/homelab/`
- Topic area for KB reference: kebab-case, e.g., `kubernetes-networking`, `helm-patterns`
- Reference filename: descriptive kebab-case, e.g., `traefik-migration.md`
- Create topic area dir under `reference/` if it doesn't exist

**3. Search for related existing notes** — Grep/Glob `~/notes/` for the same or related topics. Collect wikilinks for use in Step 4.

**4. Create the vault note** at `~/notes/<domain>/<subdirectory>/<Title>.md`:

```markdown
---
tags:
  - <domain>
  - <domain>/<subdirectory>
  - <domain>/<subdirectory>/<specific-tag>
created: <today>
updated: <today>
status: active
source: <original-url>
---

# <Title>

> **Context**: <Brief context about why this is relevant>

---

## Overview
<Main description distilled from the article>

## <Core technical sections>
<Detailed content organized by subtopic, including code blocks, YAML, commands>

## Related Areas
<Wikilinks to existing notes found in Step 3>

## References
<Links to official docs, source article, further reading>
```

**5. Create the KB reference** at `<knowledge-base>/reference/<topic-area>/<reference-filename>.md`:

```markdown
---
topic: <topic-area>
source: <original-url>
created: <today>
updated: <today>
tags:
  - <relevant-tags>
---

# <Title>

## Summary
<2-3 sentence overview optimized for machine consumption>

## Key Concepts
<Core technical details organized by subtopic — focus on facts, not narrative>

## Practical Application
<Commands, configurations, step-by-step procedures — the actionable parts>

## Decision Points
<Trade-offs, alternatives, when to use what — the judgment calls>

## References
<Links to official docs and the vault note path>
```

The reference doc must be more concise than the vault note. Strip narrative, keep facts and procedures. Optimize for OpenCode to load and reason with quickly.

**6. Update KB SKILL.md** — Read `<knowledge-base>/SKILL.md` and update:
- Directory tree in "Reference Library Structure"
- "Available Topics" table: add a new row or update an existing topic's "Contents" column

**7. Update the relevant MOC** — Add a wikilink to the new vault note in the correct section of `Tech-MOC.md`, `Work-MOC.md`, `Learning-MOC.md`, or `Life-MOC.md`. Create a new section if needed.

**8. Commit and push devops-toolkit**:

```bash
cd ~/.config/opencode/plugins/marketplaces/devops-toolkit
git add plugins/devops-toolkit/skills/knowledge-base/
git commit -m "feat(knowledge-base): add <topic-area>/<reference-filename>

Distilled from <source-url>"
git push origin main
```

**9. Report completion** — Summarize:
- Vault note path
- Reference doc path
- MOC updated
- Related notes found
- devops-toolkit commit hash
