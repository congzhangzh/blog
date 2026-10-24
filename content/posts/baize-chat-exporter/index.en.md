---
title: "BaiZe: export your AI chats to Markdown, and what keeping up with chat UIs takes"
date: 2026-10-24
summary: "BaiZe is a small Chromium extension that turns conversations from ChatGPT, Claude, Gemini and ten more sites into clean Markdown, locally. Here is what it does, and how one ChatGPT redesign broke it twice in a day."
tags: ["baize", "browser-extension", "markdown", "chatgpt", "claude", "gemini", "debugging"]
---

{{< video src="baize-trailer.mp4" poster="baize-trailer.jpg" caption="BaiZe hands back what your AIs have said, as a page you own. Generated with Gemini (Veo); the full four-shot story is in the project README." >}}

I talk to several AIs every day, and the useful parts of those conversations end up scattered
across ChatGPT, Claude, Gemini and a few more. None of them gives you a good way to keep what
was said. [BaiZe](https://github.com/congzhangzh/baize-chat-exporter) is my answer: a small
Chromium extension that exports a conversation to one clean Markdown file with one click.
Everything runs in your browser. No account, no server, no telemetry.

## The name

白泽 (Bái Zé) is the all-knowing beast of the *Shan Hai Jing* tradition. Legend says it
dictated the knowledge of all 11,520 creatures of the world to the Yellow Emperor, who had it
written down as the *Bai Ze Tu*, history's first knowledge export. The extension does the same
for your AI conversations: 汲百家所长，长出你自己的智慧, *draw from every AI, grow your own
intelligence*.

## See it work

{{< video src="demo.mp4" poster="demo.jpg" caption="Real Chrome and the real extension exporting a conversation. Only the chat site is a dummy, so no AI service is called." >}}

What you get:

- **Thirteen sites, tested:** ChatGPT, Claude, Gemini, Grok, Kimi, 豆包 Doubao,
  智谱清言 ChatGLM, 腾讯元宝 Yuanbao, Perplexity, HuggingChat, Meta AI, WhatsApp Web
  and self-hosted OpenWebUI.
- **Markdown that stays readable:** math comes back as `$...$` / `$$...$$`, code as fenced
  blocks, Claude's extended thinking folds into a `<details>` block.
- **Images included:** embedded in the file as data URIs (one file to paste anywhere), or as a
  ZIP with an `images/` folder next to the Markdown.
- **Private by design:** it only reads what the page has already rendered in your own browser.
  Extra site permissions are asked for one origin at a time, only when you add a
  self-hosted site.

## How it is built

Every site is one small adapter file, `content/adapters/<site>.js`. An adapter answers a few
questions: where is the conversation, where does each turn start, who said it, and how do I
scroll to load the rest. The framework does everything else: HTML to Markdown, images, math,
the download.

That split matters because adapters rot. Chat sites change their DOM constantly, often
silently and often per account. A class name changes, `querySelectorAll` returns an empty
list, and the user sees "no conversation found" with no clue why. So the project is built
around making those failures loud and quick to fix:

- **Two or more fallback selectors** for every critical anchor, each logging its hit count
  when you set `localStorage.AI_EXPORTER_DEBUG = '1'`.
- **A structural probe**, a snippet you paste into the page console that prints a table of how
  many nodes each candidate selector matches. Whichever selector currently works, the question
  "what still matches?" always has an answer. The probe is the real asset; the selectors are
  disposable.
- **Failure history in the source.** Every non-trivial fix adds a dated bullet to the
  adapter's header: symptom, root cause, fix. Nobody reads `git log` while debugging, but
  everybody reads the top of the file they are about to change.

## A case study: ChatGPT, twice in one day

On 28 September 2026 a bug report came in: "it does not work on ChatGPT anymore". On my test
account it worked fine.

**Same site, different page.** The reporter was on a Plus account, and ChatGPT was serving that
account a completely new app-shell layout, for both its Chat and Work modes. Free accounts
still got the old one. The probe on a Work conversation said it all: every old anchor returned
0, and the scroll container the adapter relied on was gone. Lesson one: when a bug does not
reproduce, suspect a different layout before a flaky user, and ask which account and plan.

**A scroller that runs backwards.** The new conversation list is `flex-direction:
column-reverse`. Its `scrollTop` is 0 at the *bottom* and goes negative towards the top. The
old sweep stepped up with `Math.max(0, scrollTop - step)`, which can never go below 0, so it
never moved at all. The fix was a second anchor set for the new layout and scroll bounds that
know which way is up. Verified live: a Work chat exported 16 of 16 turns, a Chat-mode chat 10
of 10, and the free-account layout still worked.

**Then it would not stop.** The same day, a second report: the export "retries forever at the
top". The cause was a fraction of a pixel. At a display scale of 1.40625, the scroller came to
rest at `scrollTop` −4989.87 while its computed bound was −4991. The check
`scrollTop > min + 1` therefore said "not at the top yet", forever, and the sweep burned its
whole time budget there. On top of that, ChatGPT loads older messages lazily, and once a fetch
missed its trigger it never retried: "Loading older messages…" sat unchanged for 60 ticks,
and only a real mouse-wheel event loaded more. The fix: a 2 px tolerance plus a "did the
scroller actually move?" test, an idle exit based on time, a small nudge away from the top
while waiting, and separate time budgets for the upward and downward passes.

None of these were hard once the evidence was on the screen. All of them would have been an
afternoon of guessing without the probe and the debug log.

## Try it

BaiZe is open source under [congzhangzh/baize-chat-exporter](https://github.com/congzhangzh/baize-chat-exporter).
A Chrome Web Store listing is being prepared. Until then:

1. `git clone https://github.com/congzhangzh/baize-chat-exporter`
2. Open `chrome://extensions` (or `edge://extensions`) and turn on **Developer mode**.
3. Click **Load unpacked** and pick the folder that contains `manifest.json`.

It works in Chrome, Edge, Brave, Opera, Vivaldi and Arc. Firefox is not supported yet.

## Who maintains what

To be clear about expectations: I maintain BaiZe for the sites and workflows I use myself,
plus the shared parts everyone depends on: the framework, the probe and the review of
contributions. **Every other site is community-owned.** DeepSeek, 通义千问 Qwen, Mistral Le
Chat, Microsoft Copilot and the rest of the wanted list in the
[README](https://github.com/congzhangzh/baize-chat-exporter#coming-soon) will arrive when
someone who uses them writes the adapter, and the same goes for keeping an existing adapter
alive on a site I don't use.

That is a smaller job than it sounds. An adapter is about 100 lines, and the person who uses a
site every day is the one who notices first when it breaks. The probe exists so that person
can find the cause without knowing the whole codebase.

## How you can help

- **Add or adopt a site.** A new site is one adapter file; the
  [contributing guide](https://github.com/congzhangzh/baize-chat-exporter/blob/main/CONTRIBUTING.md)
  walks you through it, probe first. Adopting a site means you are the one who checks it when
  it breaks.
- **Open an issue when an export breaks**, with the site, your account type and whether the
  tab was in the foreground. The probe's output table, pasted into the issue, usually points
  straight at the cause.
- **Can't write it yourself?** You can
  [sponsor a site](https://github.com/congzhangzh/baize-chat-exporter#sponsor-a-site) to move
  it up my queue. Sponsorship changes the order, not the license: BaiZe stays MIT with no paid
  features.
