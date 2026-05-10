# TenantAid: AI-Powered Tenant Rights Triage for Alameda County

> **No Poverty** | Spring 2026  
> AI Capabilities: Structured Data Extraction (Lab 2) + Text Generation (Lab 1)

---

## Problem

**Who is affected:** Rosa, 38, is a Spanish-speaking renter in Fremont who received a 3-Day Notice to Pay Rent or Quit taped to her door. She doesn't know if the notice is legally valid, whether Alameda County's just-cause eviction protections apply to her unit, or what she needs to do in the next 72 hours. She cannot afford a lawyer. The Alameda County Bar Association's tenant hotline has a 4 to 6 week callback wait. Legal Aid of Alameda County's intake form is English-only and requires knowing specific legal terminology she doesn't have.

**What specifically breaks down:** The information Rosa needs exists. Alameda County has strong tenant protections under AB 1482 and local just-cause ordinances, but it is written in legal language, scattered across county websites, and impossible to act on without knowing which law applies to her specific situation. The failure is not that resources don't exist. The failure is that a renter cannot get from "I got this notice" to "here is what I must do by when" without a lawyer.

**Why this matters locally:** Alameda County had over 22,000 eviction filings between 2021 and 2024. Fremont's renter population is majority non-white and over 30% non-English-speaking. Most tenants who lose eviction cases do so not because they have no defense, but because they didn't respond correctly or on time.

---

## AI Capability

**Which capabilities and why they fit:**

We use two AI capabilities from the labs, applied together:

**1. Structured Data Extraction (Lab 2):** The tenant pastes or describes their notice. The AI extracts a fixed schema: notice type, stated reason, dollar amount (if any), notice date, response deadline, and unit type. This mirrors exactly what we did in Lab 2, turning a messy text input into structured fields the system can act on. Without extraction, the tool cannot determine which legal protections apply.

**2. Text Generation (Lab 1):** Once the schema is populated, the AI generates a plain-language summary in the tenant's language explaining: (a) whether the notice appears legally valid, (b) which Alameda County or state protections likely apply, (c) what the tenant must do and by when, and (d) which local organization to contact first. This mirrors Lab 1's system prompt work: the output is a drafted explanation, not a legal ruling.

**Why this fits better than a simple chatbot:** A resource-finder chatbot (like the SLO example) answers "where can I get help?", useful but generic. TenantAid answers "what does THIS notice mean for ME and what do I do TODAY?", which requires first understanding the document structure before generating a response. The two-step pipeline (extract → generate) is the key design decision, justified directly by Lab 2's demonstration that schema-constrained output is more reliable and actionable than free-form text.

---

## Workflow

```
INPUT:  Tenant pastes eviction notice text or describes their situation in plain language
          ↓
STEP 1: Structured Extraction (Lab 2 capability)
        → AI extracts: {notice_type, reason, amount_owed, notice_date, deadline, unit_type, language}
          ↓
STEP 2: Rights Matching
        → Extracted fields are matched against Alameda County protection rules:
          AB 1482 (statewide just-cause), Fremont local ordinance, CDC protections
          ↓
STEP 3: Text Generation (Lab 1 capability)
        → AI drafts a plain-language action summary in the tenant's language:
          validity check / applicable protections / what to do / who to call
          ↓
OUTPUT: Structured JSON record (for caseworker intake) + plain-language summary (for tenant)
          ↓
WHO ACTS: Tenant reads summary and takes action. If urgency = HIGH (3-day notice or lockout),
          output is also flagged for a Legal Aid intake coordinator to follow up within 24 hours.
```

**Design decisions:**
- We use a two-call architecture: one call for extraction (schema-constrained JSON), one call for generation (plain language). This is more reliable than asking the model to do both at once, we learned in Lab 2 that without a schema, output format varies every run.
- The system prompt for generation explicitly instructs the model NOT to give legal advice, but to explain what the notice says and what options exist. This keeps the tool in an information role, not a legal role.
- Language detection is built into the extraction schema. If the input is in Spanish, the generation step responds in Spanish.

---

## Failure Case

**Concrete failure observed:** In our Lab 2 runs, when we submitted the homeless encampment complaint, the model extracted `department: Environmental Services`, a routing error that could have sent a person in crisis to the wrong agency. The same type of error applies here.

**Specific failure for TenantAid:** A tenant submits a notice that reads: *"You must vacate within 3 days due to lease violations."* The notice does not specify which violation. The AI extracts `reason: lease_violation` and `notice_type: 3-day-quit`. It then generates a response treating this as a standard just-cause notice under AB 1482.

**What the AI missed:** A "3-day to quit" with no opportunity to cure may be legally defective in Alameda County for many violation types, which require a "3-day to cure or quit" notice first. The tenant has a potential defense the AI did not surface.

**Real-world consequence:** Rosa, believing the notice is valid and her options are limited, does not respond or contest it. She vacates unnecessarily, losing her housing. The failure came from an ambiguous input that the model resolved in the landlord's favor rather than flagging for human review.

**Lab connection:** This parallels exactly what we observed in Lab 2: the model filled in schema fields confidently even when the input was ambiguous, producing a clean JSON output that looked correct but wasn't. Confident-looking structured output is not the same as accurate output.

---

## Oversight and Tradeoff

**Oversight position:** A human Legal Aid intake coordinator reviews any case where the extracted `notice_type` is `3-day-notice`, `unlawful-detainer`, or `lockout`, OR where the `reason` field is `ambiguous` or `multiple`, before the tenant receives the action summary. This review step is required because these are the cases where an incorrect AI output has the most severe consequence, and because our Lab 2 experience showed that the model resolves ambiguity confidently rather than flagging it.

**The one change:** We would add a mandatory ambiguity flag: if the model's confidence in the `reason` field is low, or if the notice text contains legal language that doesn't map cleanly to a known category, the output is held and routed to a coordinator rather than sent to the tenant. 

**What it costs:** This flag requires a human reviewer for an estimated 20 to 30% of submissions based on our test runs, roughly 1 additional hour of coordinator time per 10 cases. It reduces automation and adds latency (up to 24 hours) for flagged cases. The tradeoff is that the cases most likely to have tenant defenses, the ambiguous ones, get human eyes before the tenant acts on AI output. That cost is worth it for notices with 3-day deadlines, where acting on wrong information causes irreversible harm.

---

## Repository Contents

| File | Description |
|------|-------------|
| `README.md` | This file: project overview, workflow, failure case, oversight |
| `tenantaid_prototype.ipynb` | Colab notebook: extraction schema, generation prompt, test cases, outputs |


