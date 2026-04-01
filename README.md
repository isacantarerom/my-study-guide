# 🗂️ System Design Foundations

> *A collection of guides for understanding system design — built for anyone who wants to actually learn this, not just pass a test.*

These notes are written to build genuine understanding of how distributed systems work, why they're designed the way they are, and what tradeoffs are being made at every layer.

Learning something is already a success. These guides are here to accompany that process, whatever it looks like for you.

---

## Sections

**Section 1** 
> *Reccommendation:* Read them in sequence as each section builds on the last.

| Section | Topics | Guide |
|---------|--------|-------|
| **Preliminary System Design Concepts** | Abstraction, RPC, Consistency Models, Failure Models | [Read →](core-concepts/preliminary-system-design-concepts/PreliminarySystemDesignConcepts.md) |
| **Non-Functional System Characteristics** | Availability, Reliability, Scalability, Fault Tolerance | [Read →](core-concepts/non-functional-system-characteristics/NonFunctionalSystemCharacteristics.md) |
| **Back-of-the-Envelope Calculations** | Estimating servers, storage, and bandwidth | [Read →](core-concepts/back-of-the-envelope-calculations/BackOfTheEnvelopeCalculations.md) |
| **Building Blocks** | DNS, Load Balancers, Databases, Caches, Queues, CDN, and more | Coming soon... |


**Section 2**

> An [*extras*](core-concepts/extras/ExtrasIndex.md) folder worth taking a peek. 
In it you will find concepts that support the main guides without belonging to any single section.
While reading the main guides you will find thse concepts linked in them. If you are already familiarized with them feel free to skip them, but if you feel intimidated by the topic then I encourage you to give them a try.


I also highly reccommend taking a peek at [How to Approach a System Design Problem](core-concepts/extras/how-to-approach-system-design.md) independently on how familiarized you already are with the concept. It is always good to get a refresh.

---

## How to Use This Repo

Each guide is a **10–15 minute read** built around three things:

- **Core intuition** — understanding *why* something works the way it does, not just what it is
- **Real-world grounding** — concrete examples and analogies that make abstract concepts stick
- **Honest tradeoffs** — every design decision has a cost; these guides don't hide that

At the end of each guide there's a **Self-Check** section — a set of questions to answer without looking back. Don't skip it. The self-check is where the reading becomes understanding.

---

## TLDR

**→ [Start Here](core-concepts/preliminary-system-design-concepts/PreliminarySystemDesignConcepts.md)**