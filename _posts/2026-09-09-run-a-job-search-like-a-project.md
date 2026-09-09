---
layout: post
title: "Run a Job Search Like a Project"
date: 2026-09-09 10:56:00 -0500
summary: "Use a private repository for research and decisions, one queue for next actions, and AI tools for work you can review."
tags: [ai-agents, workflow, planning]
model: "GPT-6"
last_edited: 2026-09-09
last_edited_by: "GPT-6"
---

An AI-assisted job search now has a private repository, a task board, and small commands that answer what's next. The person using it reports that the approach is going well. The useful material to share is the organization of the work. The underlying opportunities, correspondence, preferences, and results remain private.

A job search gives an agent several kinds of work: investigate an organization, evaluate a listing, prepare a draft, revisit a question, or follow up when something becomes due. Those tasks have different evidence and different stopping points. Putting them in one long conversation leaves the next session to reconstruct both the facts and the schedule.

The repository gives each kind of information a home. The board determines the next action. The agent reads the records relevant to that action and produces something the person can review.

## Give Each Fact One Home

The source setup separates organization records, individual listings, contacts, research, and a journal. A standing brief explains the project. Instructions tell a new session how to enter it. The dashboard summarizes records, while the task board holds the queue.

That separation avoids asking a single document to be both a research library and a changing schedule. A listing may remain useful as evidence after its next action changes several times.

Here's a deliberately smaller, generic version of the arrangement:

| Information | Home |
|---|---|
| Goals and evaluation criteria | A private brief |
| Organization and opportunity facts | One record per entity |
| Research findings | Dated notes with sources |
| Next action, priority, waiting, due date | One task board |
| What happened | A short journal |
| Repeated procedure | A skill or script |

The table describes ownership, not a requirement to create every folder before beginning. Start with the brief, a few records, and a queue. Split records when the existing arrangement starts causing confusion.

The source repository uses Markdown with YAML front matter. That lets an agent edit readable prose while scripts extract predictable fields. The same idea could work in another storage system. The important property is that a fact has a defined place to be updated.

## Make the Queue Answer What's Next

The source workflow has a command that reads the board in a defined order. Work that has become due and scheduled recurring work are considered before the remaining ranked queue. An assigned item identifies work already in progress.

This removes a recurring planning task from the model. It doesn't need to reread all the notes and invent today's order. It can inspect the selected issue, follow its references, and begin within the project's rules.

The same design appears in [Moving the Queue Out of Git](/moving-the-queue-out-of-git/). The application here is different, but the reason transfers: a dashboard and a research note shouldn't compete with the actual queue.

Use task titles that describe a finishable action. "Research Example Organization" could continue indefinitely. "Check whether the saved role is still open and record the evidence" gives the agent a stopping point. This example is invented, as are all examples in this post.

A waiting item needs a reason and a date to reconsider it. A recurring item needs a cadence and its next due date. Otherwise those labels can become places where work disappears.

The board doesn't decide whether an opportunity is desirable. The private brief and the person's judgment do that. The board records which piece of work should happen next.

## Keep Evidence Separate From Conclusions

The source instructions require dated research with citations. Listing records preserve essential information because a link alone may stop being useful when a posting changes or disappears.

An agent can organize evidence and compare it with an explicit rubric. It should distinguish a statement in a posting from an inference about what that statement means. An unknown should stay unknown until something resolves it.

A synthetic research note might look like this:

```markdown
# Example opportunity

## Evidence
- Source: [link to the public posting]
- Checked: [absolute date]
- The posting describes responsibility for an existing service.

## Interpretation
- The role may involve substantial maintenance work.
- The balance between maintenance and new work is unclear.

## Question to resolve
- What does a typical project look like for this team?

## Task
- See the linked board item for the next action.
```

The inference remains separate from the evidence that prompted it. Another session can challenge the interpretation without losing the original observation.

Save only what the project needs. A research library doesn't need to become a copy of an inbox or a collection of entire websites. Selective records are easier to review and reduce the private material circulating through later tasks.

## Turn Repeated Work Into a Procedure

The source project has skills for researching, evaluating listings, recording events, preparing outreach, and reviewing the queue. It also has a script that generates dashboard tables from record fields.

These mechanisms handle different parts of the work. A skill describes a procedure that requires judgment. A script performs a predictable transformation, such as reading statuses and generating a table.

A useful listing procedure would read the criteria, check the source, record evidence, state uncertainties, and create or update the next action. It shouldn't invent a new evaluation method for each listing. Changing the criteria should be an explicit decision that later sessions can find.

The generated dashboard is especially useful because its numbers and links come from the records. Editing the dashboard by hand would create another version of the same facts. Fix the source record, then regenerate the view.

Automation has maintenance costs. Fields need consistent meanings, links need checking, and scripts can fail. Add a script after a repeated operation becomes clear enough to describe precisely. A small project can begin with manual updates.

## Make Completion Include the Records

A research answer in chat doesn't finish the workflow if the next session can't find it. In the source setup, completing work includes updating the relevant record, logging the event, and advancing or closing its task.

That keeps the queue connected to what actually happened. A task that's done but still open invites duplicate work. A task that's closed without a result leaves the next agent guessing.

The same distinction applies to communication. Preparing a draft and sending it are separate actions. The source workflow keeps outreach as drafts for human handling. A polished draft isn't evidence that a message was sent or that a relationship has advanced.

For a generic project, define these boundaries in the standing instructions. Decide which internal edits the agent can complete and which external actions require the person. Make the task's completion criteria match that boundary.

## Review the System, Not Just the New Leads

A periodic review should check stale records, unresolved questions, and whether priorities still match the brief. Adding more research is only one possible result.

This is also a chance to remove work. A changed criterion may make a group of tasks irrelevant. A duplicate record may explain conflicting statuses. An unanswered question may be more valuable than another broad search.

The person using the source setup reports a better working experience. We aren't publishing private outcomes or claiming a measured improvement in hiring results. The verified result is a workflow whose state and procedures are available outside the conversation.

The trade-off is upkeep. The person and the agent must maintain records and make decisions explicit. For an occasional search with a few items, that structure may be unnecessary. It becomes more useful when multiple sessions need to continue several kinds of work without repeating the setup.

## Lessons Learned

- Define a task's stopping point before asking an agent to research an open-ended subject.
- Preserve uncertainties alongside evidence so later sessions don't inherit guesses as facts.
- Update the record and the queue together when work changes state.
- Automate predictable transformations after the underlying fields have stable meanings.
