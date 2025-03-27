## Shaping: A Guide for Starting Work

Shaping turns a raw idea into a clearly defined and bounded concept. It helps us avoid getting lost in unnecessary or drawn-out efforts, and aligns our teams intention from the beginning.

It’s a practice we should apply before starting any kind of work - whether it’s a new feature, a group of tasks, or a non-technical initiative.

Before beginning any effort, we should produce a short write-up that covers the points outlined in [Shaped Output](#shaped-output).

This is a summary of Basecamp's project planning methodology [Shape Up](https://basecamp.com/shapeup), specificaly "Part 1: Shaping". It is a short guide for teams with little time to quickly get started. However, reading the entire chapter is well worth your time.

### Key Inputs

To start, we must establish:

1. **Appetite**. This is the amount of time and effort we're willing to invest. Appetite is not the estimate - it’s the boundary.([Basecamp: fixed time, variable scope](https://basecamp.com/shapeup/1.2-chapter-03#fixed-time-variable-scope)). It's informed by:
   - Motivation or urgency to solve the problem
   - How does this effort aligns with the bigger picture
   - Team availability
2. **A Commitment to Ship**. If we are going to do this thing, we must finish with something shippable\*. This:

   - Keeps the user in focus
   - Creates useful pressure
   - Forces hard decisions early

   - \* note: this also applies in scenario's where are working on something non-technical or internal. In this case "shippable" should be changed to "actionable". The output of these tasks shouldn't end with a wimper and gather dust in a corner.

### How to Shape

Shaping is **inquiry-led**: we ask questions, make notes, rearrange, and repeat.

It equally requires:

1.  design work: aninteractive process viewed from user's perspective
2.  technical literacy: knowing what is possible
3.  strategic thinking: being critical about the problem and the bigger picture

The following steps and questions can guide us in this process:

#### Set Boundaries

1. [Define the appetite](https://basecamp.com/shapeup/1.2-chapter-03#setting-the-appetite):
   1. Describe the outcome: how hungry does it make us?
   2. Consider the bigger picutre: how much time can we actually devote to this?
2. Ask questions to [narrow down the problem](https://basecamp.com/shapeup/1.2-chapter-03#narrow-down-the-problem).
   1. Don't take the first description of the problem as an accurate one. There is probably a different way to phrase it that will illuminate simpler solutions.
3. [Hammer the scope](https://basecamp.com/shapeup/1.2-chapter-03#fixed-time-variable-scope). The appetite should constrain the solution. Use this to make hard choices:
   1. Which parts of the solution are peripheral or unnecessary?
   2. Which edge cases are worth handling now, which can we ignore until later?
   3. Would a simplified or unpolished step suffice here while we focus on other more important parts?

#### Identify Risks and Rabbit Holes

Look hard to find holes, unanswered questions or unknowns that could trip us up further down the line.

1. Does this require technical work that we're unfamiliar with?
2. Are we making assumptions about how the parts fit together?
3. Have we validated a design decision?
4. Are there hard decision that we should settle in advance before we hand it off?

Risks should be noted alongside a proposed mitigations. They could be resolved simply by having a following up convo with the right people, or by putting in more research upfront

#### Sketch a Rough Solution

We need to [move at the right speed](https://basecamp.com/shapeup/1.3-chapter-04#move-at-the-right-speed): capture the design at the right level, without getting into too much detail.

The Basecamp book describes a few tools to achieve this with UX and visual elements:

- [Breadboarding](https://basecamp.com/shapeup/1.3-chapter-04#breadboarding) - borrowed from electical engineering: describe the major components and draw the wires between them.
- [Fat marker sketches](https://basecamp.com/shapeup/1.3-chapter-04#fat-marker-sketches) - draw things out in low fiedleity
- [Embedded sketches](https://basecamp.com/shapeup/1.5-chapter-06#embedded-sketches) - take a screenshot and draw over it with a marker.

When it comes to sketching technical requirements:

- [Ikusteu](https://github.com/ikusteu)'s [db refactor spec](https://github.com/librocco/librocco/discussions/603) should be taken as inspiration for describing requirements and interfaces.

### Shaped output

At the end of shaping, we should have [a pitch](https://basecamp.com/shapeup/1.5-chapter-06). This is a document that outlines:

1. **Problem** — The raw idea, a use case, or something we’ve seen that motivates us to work on this
2. **Appetite** — How much time we want to spend and how that constrains the solution
3. **Solution** — The core elements we came up with, presented in a form that’s easy for people to immediately understand
4. **Rabbit holes** — Details about the solution worth calling out to avoid problems
5. **No-gos** — Anything specifically excluded from the concept: functionality or use cases we intentionally aren’t covering to fit the appetite or make the problem tractable

Shaping is about creating clarity - not finding answers to every question. We should do enough so that we can proceed confidently.

It's important to balance the amount of prep against the size of the task: we shouldn't get bogged down over-planning a simple task, and we should be wary of embarking on an under-planned effort that could take more than a week of our collective time.
