# Awesome ePub Translator

> Translate entire ePub books with Claude — preserving the author's voice, not just the words.
>
> Built entirely in Markdown. No Python. No JavaScript. Just prompt instructions that dispatch 3 parallel Claude agents to translate a 22-chapter technical book end-to-end.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet)
![ePub 2 & 3](https://img.shields.io/badge/ePub-2%20%26%203-blue)
![Platform](https://img.shields.io/badge/platform-macOS%20%7C%20Linux%20%7C%20Windows-lightgrey)

## ⚡ Quick Start

```bash
# Install from marketplace (one time)
/plugin marketplace add ludengz/claude-plugins
/plugin install awesome-epub-translator@ludengz-plugins

# Then in any Claude Code conversation:
Translate ~/Downloads/my-book.epub to Chinese
```

That's it. Claude handles extraction, style analysis, parallel translation, and repackaging. The output lands in `output/my-book_zh.epub`. **Your original file is never modified.**

## ✨ Features

### Style Profile — the key differentiator

Before translating a single sentence, Claude reads the opening chapters and builds a profile of the book's genre, register, vocabulary level, and narrative voice. A thriller stays terse. Literary fiction keeps its cadence. A technical manual retains its precision. The profile is injected into every translation batch, so chapter 20 sounds like the same book as chapter 1.

**Example** — style profile generated from a real translation of *Node.js Cookbook, Fifth Edition*:

```
Genre: Technical reference / Programming cookbook
Tone: Professional, instructional, accessible
Voice: Second-person address ("you"), "we" for collaborative walkthroughs
Vocabulary: Keep library names, APIs, and code terms in English;
            use standard computing conventions for the target language
Section headers: "Getting ready" → "准备工作", "How it works…" → "工作原理",
                 "There's more…" → "扩展阅读"
```

You can override the auto-detected style with a custom hint: `"formal and academic"`, `"casual and conversational"`, etc.

### All features

- **Parallel 3-agent architecture** — three Claude subagents translate concurrently with size-aware file dispatch, delivering ~3x throughput over sequential tools
- **Handles unpacked ePub folders** — Apple Books and Calibre often store books as a *directory* named `something.epub`; the skill detects this and repacks before translating
- **Automatic style matching** — analyzes genre, tone, voice, and vocabulary before translating (see above)
- **Translation verification & auto-retry** — detects partially translated chapters (untranslated paragraphs, truncated files) and automatically retries them
- **Context carry-over** — terminology established in early chapters is maintained throughout the book; no more "same word, three translations" across chapters
- **Structure preservation** — all formatting, inline HTML tags, images, styles, and CSS remain intact
- **Checkpoint resumability** — translated chapters are saved incrementally; resume across conversation sessions without re-translating
- **Bilingual mode** — outputs original + translated text side by side in the same ePub with subtle CSS styling — perfect for language learners
- **ePub 2 & 3 support** — handles both `toc.ncx` and `toc.xhtml` navigation formats
- **DRM detection** — detects DRM-encrypted files and stops cleanly (see [Legal & DRM](#legal--drm))
- **Zero external services** — no API keys, no translation services beyond Claude Code itself

### What gets translated — and what doesn't

| Translated | Preserved as-is |
|------------|-----------------|
| Paragraphs, headings, lists, table cells | Code blocks (`<pre>`, `<code>`) |
| TOC entries and navigation labels | File paths, URLs, email addresses |
| Book title and description | Author names |
| Image `alt` text | `href`, `src`, `id`, `class` attributes |
| Blockquotes, figure captions | Numeric/symbolic content |

> Code stays code. `os.path.join()` will never become "操作系统.路径.加入()".

## 🎯 Quality & Consistency

**Why not just use DeepL or Google Translate?**

Traditional translation tools process text in isolation — each paragraph is translated independently with no memory of what came before. The result: "callback" becomes "回调函数" in chapter 1, "回调" in chapter 5, and "调回函数" in chapter 10.

This skill works differently:

- **Style Profile** locks in vocabulary and tone decisions before translation begins
- **Batch context** carries the last 2-3 translated paragraphs forward, maintaining narrative flow
- **Completeness verification** ensures no paragraphs are silently skipped or left untranslated
- **Human-in-the-loop** — you can pause between rounds to inspect quality before continuing

The result reads like a book, not a patchwork of machine-translated fragments.

## 📊 Real-world Benchmark

| | Details |
|---|---|
| **Book** | *Node.js Cookbook, Fifth Edition* — Packt Publishing |
| **Size** | 22 chapters · 18.5 MB · 500–1000+ lines per chapter |
| **Direction** | English → Chinese |
| **Architecture** | 3 parallel subagents × 3 rounds (9 agents total) |
| **Output** | 12.9 MB ePub — all chapters, TOC, and metadata fully translated |

> This was the stress test that drove the [v0.3.0 improvements](https://github.com/ludengz/awesome-epub-translator-skill/releases/tag/v0.3.0) — size-aware dispatch, Write-only strategy, and completeness verification were all born from this translation.

## 🌍 Supported Languages

Translate to (or from) any language Claude supports, including:

**Asian:** Chinese (Simplified & Traditional) · Japanese · Korean · Thai · Vietnamese
**European:** Spanish · French · German · Portuguese · Italian · Russian · Polish · Dutch · Swedish
**Others:** Arabic · Hindi · Turkish · Hebrew · and more

> Not limited to the list above — if Claude can read and write the language, this skill can translate it.

## 🔧 How It Works

1. **Validate & extract** — unzips the ePub, verifies structure, checks for DRM encryption
2. **Analyze style** — reads metadata and first-chapter prose to build a Style Profile capturing genre, tone, voice, and vocabulary approach
3. **Translate in parallel** — up to 3 Claude subagents work simultaneously, each assigned a balanced slice of content by file size (not naive round-robin). Context from each batch's closing sentences carries into the next, so the translation reads as one coherent voice
4. **Verify completeness** — after each round, checks every translated file for untranslated paragraphs, truncation, or missing closing tags. Incomplete files are auto-retried
5. **Translate TOC & metadata** — localizes navigation, book title, and description (author names are preserved)
6. **Repackage** — assembles a valid ePub with correct `mimetype` entry and ZIP structure

All work happens in a separate `_translation_work/` directory. Your original file is never touched. If a session ends mid-book, re-invoke the skill on the same ePub — it picks up where it left off.

## Installation

### Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- `unzip` available in your shell
- Either `zip` or Python 3 for repackaging (Python is used as fallback on Windows)

### Option A: Install from marketplace (recommended)

```bash
# Add the marketplace (one time)
/plugin marketplace add ludengz/claude-plugins

# Install the plugin
/plugin install awesome-epub-translator@ludengz-plugins

# Activate
/reload-plugins
```

### Option B: Install from source

```bash
git clone https://github.com/ludengz/awesome-epub-translator-skill.git
claude --plugin-dir ./awesome-epub-translator-skill
```

### Verify prerequisites

```bash
unzip --version
zip --version || python --version
```

See [rules/install.md](skills/awesome-epub-translator/rules/install.md) for platform-specific instructions.

## Usage

In a Claude Code conversation:

```
Translate /path/to/book.epub to Chinese
```

```
翻译 /path/to/book.epub 到日语，用双语模式
```

```
Translate my-novel.epub to Spanish with a formal, literary tone
```

### Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| Source file | Yes | — | Path to the `.epub` file |
| Target language | Yes | — | e.g., `Chinese`, `Japanese`, `zh`, `ja` |
| Output mode | No | `pure` | `pure` (translated only) or `bilingual` (side-by-side) |
| Output path | No | `output/<name>_<lang>.epub` | Custom output file path |
| Tone/style | No | auto-detect | Style hint, e.g., `"formal and academic"` |
| Pause between rounds | No | ask | `yes` to review between rounds, `no` for continuous |

### For Language Learners

Bilingual mode outputs original and translated text interleaved in the same ePub with subtle visual styling. Read Murakami in Japanese with Chinese translation on the next line — in any ePub reader.

```
Translate my-book.epub to Chinese in bilingual mode
```

## Known Limitations & Workarounds

| Limitation | Workaround |
|------------|------------|
| ~200 KB of source text per agent per round | Auto-dispatches multiple rounds; checkpoints preserve progress |
| Chapter titles rendered as images stay untranslated | Common in Calibre conversions. No OCR capability — the skill detects this and warns before starting, and still translates the TOC so navigation is in the target language |
| Text in images/SVG not translated | No OCR capability; re-create images separately if needed |
| Fixed-layout ePub may reflow poorly | Best for reflowable ePub; the skill warns about fixed-layout |
| `<ruby>`/`<rt>` annotations preserved as-is | Manual editing needed for CJK reading aids |

**Safety guarantees:**

- Your original `.epub` file is **never modified**
- All work happens in a separate `_translation_work/` directory
- You can pause, inspect quality, and resume at any time
- Delete the work directory to start over fresh

## Legal & DRM

This skill detects DRM-encrypted ePubs (via `META-INF/encryption.xml`) and refuses to process them. It is designed for ePubs you legally own and have the right to translate for personal use — DRM-free purchases, open-access books, or your own manuscripts.

## Project Structure

```
.claude-plugin/
└── plugin.json                    # Plugin manifest (name, version, description)
skills/
└── awesome-epub-translator/       # Skill package
    ├── SKILL.md                   # Main skill definition (10-step workflow)
    ├── references/
    │   ├── translation-prompt.md  # Translation prompt template
    │   └── epub-structure.md      # ePub format quick reference
    └── rules/
        └── install.md             # Platform-specific install instructions
```

## See Also

- [Claude Code](https://claude.ai/code) — the CLI platform this skill runs on
- [Calibre](https://calibre-ebook.com/) — desktop ebook manager with format conversion and basic translation plugins
- [ePub specification](https://www.w3.org/publishing/epub/) — W3C ePub 3 standard reference

## License

MIT
