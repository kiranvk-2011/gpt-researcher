# PR #1737 -- Contribution to GPT Researcher

---

## LinkedIn Post (Copy-paste this)

Sometimes the hardest bugs to find are the ones where the code does nothing wrong -- technically. It just does nothing at all.

We ran into this with GPT Researcher, the open-source deep research tool. The report-writing phase would hang for 2.5 minutes with zero output. No error. No crash. Just silence. The fix? Two lines of Python. But finding those two lines took tracing through the full streaming architecture, from LLM provider to stdout.

Turns out, some LLM providers (especially thinking models) send hundreds of empty string chunks before any real content. The existing code checked `if content is not None` -- which passes for `""`. So the buffer accumulated nothing, the flush trigger never fired, and stdout stayed silent.

PR merged same day. Now shipping to every GPT Researcher installation.

**Read the full story:** [PR #1737 Contribution Story](https://github.com/kiranvk-2011/gpt-researcher/blob/fix/streaming-empty-chunks-flush/PR-1737-journey.md#the-full-story)

**View the PR:** [assafelovic/gpt-researcher#1737](https://github.com/assafelovic/gpt-researcher/pull/1737)

#OpenSource #Python #AIResearch #GPTResearcher #LLM #Streaming #SoftwareEngineering #DeepResearch

---

## The Full Story

### How it started

We'd set up GPT Researcher as a self-hosted deep research tool on our VPS, wired up to a zero-cost LLM provider (Alibaba Cloud's Bailian platform, using GLM-5). The setup worked -- queries went through, reports came back. But there was a problem. During the report-writing phase, the terminal would go completely quiet for about 2.5 minutes. No progress. No streaming output. Nothing. Then suddenly, the entire finished report would dump to stdout all at once.

For a tool designed to generate multi-thousand-word research reports, watching a blank screen for minutes with no idea whether something is working or broken isn't a great experience. We wanted to understand why.

### Tracing the streaming pipeline

GPT Researcher's architecture splits work into phases: question generation, web search, source scraping, and finally report writing. The first phases are quick. The report-writing phase is where the LLM does the heavy lifting -- synthesizing scraped content into a structured research report. This is also where streaming output matters most, because you're waiting the longest.

The streaming pipeline turned out to be straightforward once you traced it end to end:

```
LLM provider (GLM-5 via OpenAI-compatible API)
  -> async generator yields content chunks
  -> stream_response() in base.py buffers chunks into paragraphs
  -> on "\n", flushes paragraph to stdout via _send_output()
  -> _send_output() calls sys.stdout.write() + sys.stdout.flush()
```

The key function was `stream_response()` in `gpt_researcher/llm_provider/generic/base.py`. It accumulates content chunks into a `paragraph` string, and when it encounters a newline character, it flushes the paragraph to stdout and resets. Simple enough.

### Finding the root cause

Here's the thing about thinking models like GLM-5: they don't start emitting content immediately. The model thinks first -- sometimes for 10-15 seconds -- and during that time, the API sends keep-alive chunks. These chunks have `content: ""` (empty string), not `content: None`. Hundreds of them.

The original code had this check:

```python
if content is not None:
    paragraph += content
```

Empty string passes that check. So `paragraph += ""` ran hundreds of times, accumulating nothing. The `"\n"` flush trigger never fired because there was no content containing a newline. The paragraph buffer stayed empty while the model was thinking, and by the time real content started arriving, the entire report was buffered internally and only flushed when the stream completed.

There was a second issue too. `_send_output()` called `sys.stdout.write()` but never called `sys.stdout.flush()`. In a pipe or non-TTY context (which is how we ran it from our wrapper script), Python buffers stdout by default. Without an explicit flush, even the final dump might not appear until the process exits.

### The fix

Two changes, both in `base.py`:

**1. Skip empty chunks in `stream_response()`:**
```python
# Before:
if content is not None:
    paragraph += content

# After:
if content is not None and content != "":
    paragraph += content
```

**2. Flush stdout in `_send_output()`:**
```python
# Before:
sys.stdout.write(output)

# After:
sys.stdout.write(output)
sys.stdout.flush()
```

That's it. Two lines changed. The first prevents empty chunks from silently passing through the accumulator. The second ensures output actually reaches the terminal in real time.

### Testing it

After applying the fix locally, we ran the same research query that had previously shown the 2.5-minute silence. This time, streaming output appeared within seconds of the report-writing phase starting. Paragraphs flowed to stdout as they were generated, exactly as you'd expect. The total report time didn't change -- it was never slow, just silent.

We also verified that the fix was safe for all providers, not just GLM-5. The empty-string check is a no-op for providers that don't send empty chunks, and the stdout flush is universally correct behavior for streaming output.

### Going upstream

We forked the repo, created a clean branch, committed the fix, and opened PR #1737 with a clear explanation of the root cause and the streaming architecture trace. We also filed issue #1738 to document the bug for anyone else hitting it.

The PR was reviewed and merged the same day by **Assaf Elovic**, the project creator. Clean, no back-and-forth needed. Sometimes the best contributions are the smallest ones -- when the diagnosis is thorough and the fix is obviously correct, the review process reflects that.

### What made this interesting

This bug is a good example of something that's technically not a bug in any single component. The LLM provider is doing nothing wrong by sending empty chunks. The `is not None` check is a reasonable null guard. Python's stdout buffering is documented behavior. Each piece is correct in isolation. The failure only emerges from their interaction -- empty strings that aren't null, passing through a buffer that expects content, writing to a stream that doesn't flush.

Finding it required tracing the full pipeline end to end, not just looking at the error (there was no error) or the symptoms (silence isn't a stack trace). It's the kind of bug that rewards patience over cleverness.

### The tool behind the work

Same as with our previous open-source contributions -- **Claude** was the AI coding partner throughout this process. From tracing through the streaming architecture, to identifying the empty-string vs None distinction, to verifying the fix was safe across providers. The combination of human intuition ("something is wrong with streaming") and AI-assisted code archaeology ("here's exactly where the empty chunks slip through") made the diagnosis fast and the fix precise.

### What ships now

PR #1737 merged on April 16, 2026. The fix is now part of GPT Researcher for everyone. Anyone running GPT Researcher with thinking models, reasoning models, or any LLM provider that sends empty keep-alive chunks during generation will now see streaming output in real time instead of a silent wait followed by a wall of text.

Two lines. One merged PR. Better experience for every user.

---

**PR:** [assafelovic/gpt-researcher#1737](https://github.com/assafelovic/gpt-researcher/pull/1737) | 2 lines changed in `base.py` | Merged by [@assafelovic](https://github.com/assafelovic)

**Skills demonstrated:** End-to-end streaming architecture tracing, Python async generators, LLM provider behavior analysis, open-source contribution, AI-assisted debugging

**Looking to solve hard infrastructure problems together?** [Reach out.](https://www.linkedin.com/in/kiranvk2011/)
