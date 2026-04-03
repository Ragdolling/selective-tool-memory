# Selective Tool Output Memory for Agents: A Small Experiment

This note comes from a tool-heavy AI agent I built inside the Unreal Editor.

The problem was not chat history. It was tool output.

Large `search`, `list`, and `inspect` results were often useful for one intermediate step, then kept polluting long-term context for many later turns.

## What I changed

I added two memory tools:

- `tool_output_archive`
- `tool_output_read`

The design was intentionally narrow.

I did **not** try to summarize whole conversations.
I only targeted already-consumed large tool outputs.

The behavior was:

- `tool_output_archive` moves a large tool result out of long-term context and leaves behind a short `replacement_output`
- `tool_output_read` can temporarily expose the original raw output again by `tool_ref`
- `tool_output_read` does **not** restore the raw output back into long-term history

The most important rule was that the memory tools themselves should not become new long-term noise.
They were only made visible to the immediately following continuation, then cleared.

I also kept the policy conservative:

- archive only discovery-style outputs
- archive only clearly large outputs, roughly above 4 KB
- only allow archive on a recent window of tool results
- do not force the model to archive

## What I measured

I ran real tasks for two days, then scanned the saved session logs.

At the time of writing, the sample was:

- 60 sessions
- 21 archive calls
- 25 archived source tool outputs
- 0 read calls

Across all 60 sessions, estimated long-term context went from:

- `1,047,789` tokens without archive
- to `856,353` tokens with archive

That is a reduction of:

- `191,436` tokens
- `18.27%` overall

Looking only at the 12 sessions that actually used archive:

- `493,435 -> 301,999`
- `191,436` tokens saved
- `38.8%` reduction
- about `15,953` tokens saved per archive-using session on average

Best single session:

- `93,606 -> 34,837`
- `58,769` tokens saved
- `62.78%` reduction

## What did not happen

This is the part I think matters most.

I am **not** claiming this is fully validated.

What I can say:

- when archive was used, it usually saved a meaningful amount of long-term context
- I did not observe an obvious drop in task completion quality

What I cannot say yet:

- I did not run a proper benchmark
- I did not quantify task success rate before vs after
- I cannot cleanly isolate cache effects from this change alone

There was also one interesting negative result:

- `tool_output_read` was never used in the observed runs

That may simply mean the current archive policy is conservative enough that the model rarely needed to revisit raw output.
It does not necessarily mean the mechanism is useless.

Another limitation:

- archive was intentionally under-used

In these 60 sessions, only 12 used it at all.
That was not because I was trying to maximize archive frequency.
It was because I did not want to push the model too hard and risk incorrect archiving, distorted summaries, or cache-unfriendly history changes.

So the current bottleneck is less about "can the model trigger archive reliably" and more about where the safe archive boundary really is.

## Why I think this is still interesting

Most context-control discussions in agents drift toward generic summarization.

This experiment suggests a narrower target may be more useful:

- do not summarize everything
- aggressively manage tool outputs first
- keep raw evidence recoverable
- keep the recovery path transient, not persistent

That shape felt much more controllable than free-form conversation memory, especially when the bigger risk is not under-archiving but archiving the wrong thing.

## Why there is no code here

This is not just "add two tools".

It changes the context assembly path:

- how tool results are stored
- how archived outputs are represented
- what enters long-term context
- what is only injected for the next continuation

A few isolated functions would make the change look much simpler than it really is.

## If you want to implement the same idea

You can give the following prompt to an AI coding assistant and let it adapt the pattern to your own agent architecture:

```text
Implement two memory tools in the current agent project:

1. tool_output_archive
2. tool_output_read

Do not summarize whole conversations.
Only handle large completed tool outputs that have already been consumed.

Requirements:

- archive takes one or more short tool_ref values, not long call_id values
- after archive, raw tool output must stop entering long-term context
- long-term context should keep only a short replacement_output
- raw output and attachments must still be stored internally
- archived results should carry an explicit archived marker when exposed to the model
- tool_output_archive itself must not enter long-term context; it should only be visible to the immediately following continuation

- read takes tool_ref values
- read should expose the raw archived output only to the next continuation
- read must not restore raw output into long-term history
- tool_output_read itself must not enter long-term context

Please inspect and update:

- session history data structures
- tool result persistence
- provider/context assembly logic
- long-term vs transient context injection

Add tests for:

- long-term context really gets shorter after archive
- read does not restore long-term history
- memory tool visibility lasts only one continuation
- archived results are explicitly marked when exposed to the model
```

## Current takeaway

This is not a finished solution.

But it is enough evidence for one claim:

for tool-heavy agents, a lot of context growth is really tool-output growth.

Managing that directly may be more useful than trying to summarize everything.
