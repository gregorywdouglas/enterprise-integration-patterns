# Contributing to the Enterprise Integration Patterns Library

Thank you for contributing to this pattern library. The goal is to maintain a high-quality, practitioner-focused reference — not a documentation dump. Every contribution should add concrete value to integration architects working on real Azure implementations.

---

## Who Should Contribute

This library is intended for senior integration architects, platform engineers, and solution architects with hands-on Azure integration experience. Contributions from practitioners who have implemented a pattern in production environments are strongly preferred over purely theoretical additions.

---

## Types of Contributions

- **New patterns** — A clearly named, reusable solution to a recurring integration problem
- **Pattern refinements** — Corrections, clarifications, or additions to existing patterns based on implementation experience
- **Azure component updates** — Corrections or additions to component reference documents reflecting service changes
- **Observability standards updates** — Refinements to correlation, tracing, or exception handling guidance
- **Structural improvements** — Navigation, formatting, or cross-linking improvements

---

## Before You Contribute

1. **Search first.** Check whether a pattern already exists or is being worked on. Open issues may already track the addition you have in mind.
2. **Open an issue before large additions.** For new pattern files or significant restructuring, open an issue to discuss scope, naming, and placement before writing content.
3. **Distinguish patterns from practices.** A pattern is a named, reusable solution to a recurring problem. Not every integration decision qualifies. If you are uncertain, open a discussion issue first.

---

## Pattern Template

Every new pattern file must follow this structure exactly. Do not add or remove top-level sections without prior discussion.

```markdown
# [Pattern Name]

## Problem

[One to three paragraphs describing the recurring problem this pattern addresses. Be specific about the integration context — what systems are involved, what breaks without this pattern, and why the naive approach fails at scale or under operational conditions.]

## Solution

[Description of the pattern's core mechanism. What does it do structurally? How does data or control flow through it? Include a conceptual diagram description if useful. Avoid prescribing a single technology here — describe the solution abstractly first, then move to implementation.]

## When to Use

[Bullet list of conditions under which this pattern applies. Include both positive signals (use when...) and negative signals (avoid when...). Be honest about tradeoffs.]

- Use when ...
- Use when ...
- Avoid when ...
- Avoid when ...

## Azure Implementation

[Concrete implementation guidance using specific Azure services. Include:]
- Which Azure services play which roles
- Relevant configuration, policy XML, ARM/Bicep snippets, or pseudocode
- Authentication and networking considerations
- Relevant service limits or quotas that affect the pattern

### [Sub-section per implementation aspect as needed]

## Key Considerations

[Operational, security, scalability, and observability concerns specific to this pattern. Include:]
- Failure modes and mitigations
- Observability requirements (correlation IDs, tracing)
- Cost implications at scale
- Common implementation mistakes

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
```

---

## Submitting a New Pattern

### Step 1: Fork and Branch

```bash
git clone https://github.com/your-org/enterprise-integration-patterns.git
cd enterprise-integration-patterns
git checkout -b pattern/your-pattern-name
```

Use the naming convention `pattern/kebab-case-name` for pattern additions, `fix/description` for corrections, and `docs/description` for documentation improvements.

### Step 2: Create the Pattern File

Place the file in the appropriate subdirectory:
- `patterns/api/` — Synchronous API patterns
- `patterns/event-driven/` — Asynchronous and event-driven patterns
- `patterns/hybrid/` — Hybrid and cross-boundary patterns

Use PascalCase with hyphens for the filename: `My-New-Pattern.md`

### Step 3: Update the Pattern Index

Add an entry to the pattern table in `README.md` under the appropriate category. Use the same relative link format as existing entries.

### Step 4: Self-Review Checklist

Before submitting, confirm:

- [ ] The pattern file follows the standard template structure exactly
- [ ] The **Problem** section describes a real, recurring problem — not a hypothetical
- [ ] The **Azure Implementation** section references specific Azure service names and configurations
- [ ] The **When to Use** section includes both positive and negative signals
- [ ] The **Key Considerations** section covers failure modes and observability
- [ ] The pattern is referenced in `README.md`
- [ ] The file uses consistent Markdown formatting (no raw HTML, consistent heading levels)
- [ ] No proprietary or confidential implementation details from client engagements are included
- [ ] The file attribution footer is present

### Step 5: Open a Pull Request

Open a PR with:
- **Title:** `Add pattern: [Pattern Name]` or `Update: [Pattern Name] - [brief description]`
- **Description:** 2–4 sentences explaining what you added or changed and why
- **Labels:** Apply appropriate labels (`new-pattern`, `correction`, `azure-component`, `observability`)

---

## Review Process

All PRs are reviewed for:

1. **Technical accuracy** — Does the pattern reflect how Azure services actually behave? Are configuration examples correct?
2. **Template compliance** — Does the file follow the required structure?
3. **Tone and audience** — Is the content written for senior practitioners? Avoid marketing language, vague recommendations, and hedging.
4. **Completeness** — Is the pattern substantive enough to be actionable? Stubs will not be merged.

Reviews are conducted by the library maintainers. Expect a response within 5 business days. Reviewers may request revisions before approval.

---

## Code of Conduct

All contributors are expected to interact professionally and constructively. This is a technical library maintained by practitioners for practitioners. Debates about technical approach are welcome; personal criticism is not.

Specifically:
- Critique ideas and implementations, not contributors
- Be direct about technical disagreements — vague approval is not helpful
- Assume good faith in contributions from others
- Maintain confidentiality — do not include client names, proprietary data, or internal system names from your engagements

Persistent violations of professional conduct will result in removal from the contributor list.

---

## Questions

Open a GitHub Discussion or issue tagged `question`. Do not open PRs to ask questions.

---

*This contributing guide is maintained alongside the pattern library. If you find it unclear or incomplete, submit a PR to improve it.*
