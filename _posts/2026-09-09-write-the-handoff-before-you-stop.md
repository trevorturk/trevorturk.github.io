---
layout: post
title: "Write the Handoff Before You Stop"
date: 2026-09-09 10:55:00 -0500
summary: "A focused restart prompt can make a long session easier to leave overnight. What it preserves, what it costs, and when compaction is enough."
tags: [ai-agents, workflow, context, tokens]
model: "GPT-6"
last_edited: 2026-09-09
last_edited_by: "GPT-6"
---

A long engineering session ended with a change of direction. We had spent the conversation separating experiments from a large implementation. The next session needed to reconsider the remaining work, starting with a smaller baseline. Carrying the whole conversation forward would preserve plenty of history that no longer defined the task.

The immediate concern was leaving the session overnight. If the session became unavailable, or its prompt cache expired, would starting again mean processing a large amount of context? Would compacting before bed solve that?

We asked the agent for a prompt to start a new session. It produced a focused handoff: current state, decisions, deferred options, references, constraints, and the first planning task. The useful part was the selection. Tomorrow's agent received an explicit assignment instead of having to infer it from the conversation.

This is worth treating as a repeatable workflow. I'll call it a session handoff, a descriptive name rather than a claim that we've found a new named technique. It's closely related to writing a checkpoint or handing work to another person.

## Separate the Three Things You Can Lose

A saved conversation, the model's working context, and a provider's prompt cache serve different purposes. Confusing them makes it hard to decide what to preserve.

The saved conversation is the record the application may let you reopen. Working context is the material supplied to the model for its next response. A readable transcript doesn't establish how much of it the application will supply next time.

Prompt caching reuses computation for matching prompt prefixes. Its availability depends on retention and routing, among other factors. Keeping a terminal open doesn't itself establish that tomorrow's request will hit the cache. A cache miss also doesn't mean the saved conversation has disappeared. OpenAI documents these mechanisms in its [prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching).

A handoff addresses the information needed to restart. It doesn't keep a cache warm, extend a session's lifetime, or restore an expired process. Its potential token benefit comes from making the next task require less input and less repeated investigation.

## When Compaction Is Enough

Compaction reduces the context carried into subsequent turns. OpenAI's [API compaction guide](https://developers.openai.com/api/docs/guides/compaction) describes a compacted representation that preserves key state using fewer tokens. That representation can include opaque content; it isn't necessarily a prose summary you can inspect. Individual applications decide how to expose compaction and session resumption.

If you're continuing the same task in a resumable session, compaction may be enough. There may be no reason to write a second summary, review it, and start elsewhere. A fresh session can lose useful details and spend more tokens rediscovering them.

The handoff becomes more useful when the assignment changes. Our source conversation had reached that boundary. The next step was a smaller planning pass, with a different interpretation of which experiments belonged together. A concise assignment could state that change directly.

It also helps when switching tools or preparing work for another agent. The prompt can point to durable records without depending on a particular application's saved session format.

These approaches can be combined. Save the decisions and a short next-task brief, then compact and resume if that remains convenient. The handoff provides a fallback. It doesn't require abandoning a usable conversation.

## Write an Assignment, Not a Transcript

The source handoff distinguished completed work from the preferred direction. It preserved options the user wanted to explore later. It also told the next agent to verify the current state before relying on the recorded details.

That last instruction matters because a handoff ages as soon as work continues elsewhere. A statement that tests passed describes a particular revision. It doesn't establish that today's checkout passes them.

Here's a synthetic prompt for producing a handoff. It contains none of the source project's identifiers or implementation details:

```text
Prepare a handoff for a fresh session tomorrow.

Include:
- The next task and its intended result.
- Confirmed decisions, including why we made them.
- Completed work, with references and verification scope.
- Options deferred, rejected, or still undecided.
- The few files or issues needed to begin.
- Constraints and actions that still need approval.
- The first concrete action for the next session.

Distinguish observations from proposals. Mark facts that need
rechecking. Point to durable records instead of copying their
contents. Omit investigation history that won't change the next task.

Return a pasteable kickoff prompt. Don't perform the next task.
```

The deferred category prevents a common misunderstanding: postponing an option can look like rejecting it after a summary loses the distinction. Preserving that decision can matter more than preserving several pages of implementation discussion.

Review the handoff before ending the session. Check that tentative ideas haven't become requirements, completed work has evidence, and the first action matches what you actually want tomorrow.

## Put Decisions Somewhere That Outlives the Prompt

A restart prompt is a convenient entry point. A project record is a better home for decisions that several sessions will need.

For an ongoing initiative, maintain a short decision checklist in its existing issue or plan. Distinguish wanted next, consider later, rejected, and completed. Link completed items to their evidence. The source conversation proposed this arrangement; it didn't demonstrate that the checklist had already been created.

The handoff can then refer to the checklist and explain what to do with it. That keeps tomorrow's prompt smaller and gives later sessions a place to check for changes.

Avoid copying the queue into the handoff. A copied priority list becomes a competing schedule. The same concern motivated [Moving the Queue Out of Git](/moving-the-queue-out-of-git/): the next action needs one authoritative home.

For a small task, a saved prompt may be sufficient. There's little benefit in creating a permanent document for a brief that will be obsolete after one session. Keep the durable decisions; discard the temporary assignment when it has served its purpose.

## Measure the Whole Restart

The original conversation included cumulative token usage. It didn't include a comparison between resuming, compacting, and starting from the handoff. We can't turn that session total into a savings claim.

The relevant comparison includes preparing the handoff, tomorrow's input, and any extra investigation caused by omissions. Where the tool exposes it, record uncached input, cached input, output, and cache-write charges separately. Use the applicable billing or usage rules rather than treating those categories as interchangeable.

Compare the same next task from equivalent project states. Record whether the agent reaches a useful result, how much it rereads, and whether it misses a decision. A shorter initial prompt that causes a long rediscovery pass may lose the comparison.

For an overnight break, the practical question is how much information tomorrow's task needs if cache reuse is unavailable. Both compaction and a handoff can reduce that amount. Neither should be described as guaranteed cheaper without a measurement.

## What We Can Claim

The source shows that a long conversation could produce a concrete restart assignment. The user reports finding that workflow valuable. We haven't measured its token savings or established that it beats compaction.

The accepted cost is an extra writing and review step before stopping. The benefit is a readable statement of what comes next, especially after the project changes direction. If the next task is unchanged and the session resumes well, that extra step may add little.

## Lessons Learned

- Compare restart methods on the same task, including the work required to recover omitted context.
- Preserve the reason for deferring an option when a later agent might mistake it for rejection.
- Attach verification claims to the revision they describe.
- Keep a temporary assignment small by linking to maintained project records.
