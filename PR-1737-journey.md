# PR #1737 -- Contribution to GPT Researcher

---

## LinkedIn Post (Copy-paste this)

Sometimes the hardest bugs to find are the ones where the code does nothing wrong -- technically. It just does nothing at all.

We ran into this with GPT Researcher, the open-source deep research tool. The report-writing phase would hang for 2.5 minutes with zero output. No error. No crash. Just silence. The system was effectively relying on the LLM finishing its entire response before showing you anything -- the exact opposite of what streaming is supposed to do.

The root cause? Some LLM providers send hundreds of empty string chunks before real content arrives. The code checked `if content is not None` -- which passes for `""`. So the streaming buffer accumulated nothing, never flushed, and stdout stayed dark until the whole report was done.

Two lines of Python. One merged PR. Streaming output now works for every LLM provider.

**Read the full story:** [PR #1737 Contribution Story](https://github.com/kiranvk-2011/gpt-researcher/blob/fix/streaming-empty-chunks-flush/PR-1737-journey.md#the-full-story)

**View the PR:** [assafelovic/gpt-researcher#1737](https://github.com/assafelovic/gpt-researcher/pull/1737)

#OpenSource #Python #AIResearch #GPTResearcher #LLM #Streaming #SoftwareEngineering #DeepResearch

---

## The Full Story

### A quick primer: what streaming actually means

When you ask an LLM to write something, there are two ways to get the response back. The simple way is to wait: you send a prompt, the model processes it, and when it's completely done, you get the whole thing at once. That's fine for short answers, but for a 3,000-word research report, you're staring at a blank screen for minutes wondering if something broke.

The better way is streaming. The model sends you tokens as it generates them -- word by word, sentence by sentence. You see the report taking shape in real time. It's the same principle as watching a web page load progressively instead of waiting for every image to finish before seeing anything. The total time is identical, but the experience is fundamentally different. With streaming, you know the system is working. Without it, you're trusting that silence means progress, not failure.

Most modern LLM APIs support streaming natively. The model returns an async stream of small chunks, and the application is responsible for collecting them and presenting them to the user as they arrive. That's the contract. But contracts have edge cases.

### How GPT Researcher's streaming was supposed to work

GPT Researcher is an open-source tool that automates deep research -- you give it a question, and it generates search queries, scrapes web sources, and synthesizes everything into a structured report. The architecture splits this into phases: question generation, web search, source scraping, and finally report writing. The first few phases are quick. The report-writing phase is where the LLM does the heavy lifting, and it's where you wait the longest.

The streaming pipeline was straightforward:

```
LLM provider (OpenAI-compatible API)
  -> async generator yields content chunks
  -> stream_response() in base.py buffers chunks into paragraphs
  -> on "\n", flushes paragraph to stdout via _send_output()
  -> _send_output() calls sys.stdout.write()
```

`stream_response()` was the heart of it. As chunks arrived from the LLM, it concatenated them into a `paragraph` buffer. When it hit a newline character, it flushed the paragraph to stdout and reset the buffer. A simple, reasonable design: accumulate until you have a complete paragraph, then display it.

The implicit assumption was that chunks would contain actual text. And for many LLM providers, they do. But not all of them.

### What actually happened in practice

We'd set up GPT Researcher on our VPS, wired up to Alibaba Cloud's Bailian platform using GLM-5 -- a zero-cost thinking model. The setup worked. Queries went through, reports came back. But during the report-writing phase, the terminal went completely silent for about 2.5 minutes. No streaming. No progress indicator. Nothing. Then the entire finished report would dump to stdout all at once.

If you've ever sat watching a blank terminal wondering whether a long-running process is working or frozen, you know the feeling. For a tool designed to generate multi-thousand-word reports, that experience was basically: "trust that it's working, or kill it and try again." The system was effectively relying on the full completion of the LLM response as the only signal that anything had happened. Streaming was implemented in the code, but in practice, it wasn't streaming at all.

### Finding the root cause

The critical insight came from understanding how thinking models behave at the API level. Models like GLM-5 don't start emitting content tokens immediately. They have a "thinking" phase -- sometimes 10-15 seconds -- where the model is reasoning internally before it begins writing. During this phase, the API keeps the connection alive by sending chunks. But these chunks have `content: ""` (empty string), not `content: None`. Hundreds of them.

The original code had this check:

```python
if content is not None:
    paragraph += content
```

Empty string is not None. So `paragraph += ""` executed hundreds of times, accumulating exactly nothing. The newline-based flush trigger never fired because there was no content containing a newline. The paragraph buffer stayed empty while the model thought, and when real content finally started flowing, it buffered internally until the stream completed.

There was a second, compounding issue. `_send_output()` called `sys.stdout.write()` but never called `sys.stdout.flush()`. When you run a Python script from a wrapper (pipe, cron, subprocess -- any non-TTY context), Python switches to block-buffered stdout. Without an explicit flush after each write, output accumulates in Python's internal buffer and only appears when the buffer fills up or the process exits. So even when paragraphs did eventually get flushed by the streaming logic, they might not reach the actual terminal until much later.

The two issues combined into a perfect silence: empty chunks prevented the streaming logic from firing, and missing flushes prevented whatever output did get through from being visible. The system fell through to its implicit fallback behavior -- dump everything at process completion -- which happened to work, just with zero real-time feedback.

### The fix

Two changes, both in `gpt_researcher/llm_provider/generic/base.py`:

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

That's it. The first change prevents empty chunks from silently passing through the accumulator -- when actual content arrives, the paragraph buffer is clean and ready, and the newline flush trigger works as designed. The second change ensures that every paragraph reaches the terminal immediately, regardless of how the Python process was invoked.

Together, they turn GPT Researcher's streaming from "technically implemented but functionally broken for thinking models" into "works correctly for every LLM provider."

### What changes for users

The impact depends on how you use GPT Researcher.

If you run it from a terminal directly and only use models like GPT-4 that emit content from the first chunk, you might never have noticed the bug. Those models don't send empty keep-alive chunks, so the streaming pipeline happened to work correctly by accident.

If you use thinking models (GLM-5, DeepSeek, QwQ, or any reasoning model that pauses before generating), the difference is night and day. Before: 2-3 minutes of silence, then a wall of text. After: streaming output begins as soon as the model starts writing, paragraph by paragraph, exactly as the architecture intended.

If you invoke GPT Researcher from scripts, wrappers, or CI pipelines (non-TTY contexts), the stdout flush fix means output appears in real time instead of being buffered until process exit. This matters for logging, for progress monitoring, and for not killing a process because it looks hung when it's actually working fine.

The total report generation time doesn't change -- the model was never slow, just silent. The fix makes the existing work visible.

### Going upstream

We forked the repo, created a clean branch, committed the two-line fix, and opened PR #1737 with a detailed explanation: the streaming architecture trace, the root cause analysis (empty string vs None), the thinking-model behavior, and why the fix is safe for all providers (the empty-string check is a no-op for providers that don't send empty chunks).

We also filed issue #1738 to document the bug for anyone else hitting the same symptom -- silent report generation with no errors.

The PR was reviewed and merged the same day by **Assaf Elovic**, the project creator. No back-and-forth, no revision requests. When the diagnosis is thorough and the fix is obviously correct, the review process reflects that. That's one of the things I genuinely appreciate about well-run open-source projects -- a clean contribution gets a clean merge.

### What made this interesting

This bug is a good example of something that's technically not a bug in any single component. The LLM provider is doing nothing wrong by sending empty chunks -- it's keeping the connection alive during the thinking phase. The `is not None` check is a perfectly reasonable null guard. Python's stdout buffering is documented, expected behavior. Each piece is correct in isolation. The failure only emerges from their interaction: empty strings that aren't null, passing through a buffer that expects content, writing to a stream that doesn't flush.

It's also a case where the system had a silent fallback that masked the problem. The report still got generated and delivered. Nothing crashed. No error was thrown. The streaming just quietly didn't stream. In a way, the system's resilience -- its ability to "work" even when streaming broke -- is exactly what made the bug hard to notice and easy to live with. You had to care about the user experience specifically to even recognize it as a problem.

### The tool behind the work

Same as with our previous open-source contributions -- **Claude** was the AI coding partner throughout. From tracing the async generator pipeline, to spotting the `is not None` vs `!= ""` distinction in the streaming accumulator, to verifying that the fix was safe across provider behaviors. The pattern we've settled into works well: human intuition identifies that something is wrong ("this should be streaming but isn't"), AI-assisted archaeology finds exactly where and why, and the human makes the judgment call on the minimal correct fix. Neither part works as well alone.

### What ships now

PR #1737 merged on April 16, 2026. The fix is now part of GPT Researcher for everyone. Anyone running it with thinking models, reasoning models, or any LLM provider that sends empty keep-alive chunks during generation will see streaming output in real time instead of a silent wait followed by a wall of text.

Two lines. One merged PR. Streaming that actually streams.

---

**PR:** [assafelovic/gpt-researcher#1737](https://github.com/assafelovic/gpt-researcher/pull/1737) | 2 lines changed in `base.py` | Merged by [@assafelovic](https://github.com/assafelovic)

**Skills demonstrated:** End-to-end streaming architecture tracing, Python async generators, LLM provider behavior analysis, open-source contribution, AI-assisted debugging

**Looking to solve hard infrastructure problems together?** [Reach out.](https://www.linkedin.com/in/kiranvk2011/)
