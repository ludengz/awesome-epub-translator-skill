---
name: awesome-epub-translator
description: >
  Translate an ePub file from one language to another with high-quality AI translation.
  Preserves formatting, inline HTML tags, images, and styles.
  Supports pure translation and bilingual (original + translated) output modes.
  Use when the user asks to: translate an ePub, translate an e-book, translate a book,
  翻译电子书, 翻译ePub, convert ePub to another language.
---

# ePub Translator

Translate ePub files between languages while preserving the original structure, formatting, and visual appearance.

## Parameters

The user provides these when invoking the skill:

| Parameter | Required | Description | Example |
|-----------|----------|-------------|---------|
| Source file | Yes | Path to the .epub file | `/path/to/book.epub` |
| Target language | Yes | Language to translate into | `Chinese`, `Japanese`, `zh`, `ja` |
| Output mode | No (default: pure) | `pure` = translated only, `bilingual` = original + translated | `bilingual` |
| Output path | No (default: `output/<name>_<lang>.epub`) | Custom output file path | `/path/to/book_zh.epub` |
| Tone/style | No (default: auto-detect) | Style hint for translation | `"formal and academic"` |
| Pause between rounds | No (default: ask) | Whether to pause after each parallel round for confirmation | `yes`, `no` |

If the user doesn't specify all required parameters, ask for them before proceeding.

If `Pause between rounds` is not specified, ask the user after confirming the style profile (end of Step 5.5):
> "This book has N chapters to translate. Would you like me to pause after each round for confirmation, or translate everything in one go? (Pausing lets you check progress; continuing is faster for large books.)"

## Prerequisites

- `unzip` must be available in the shell
- For repackaging: either `zip` command OR Python 3 (used as fallback)
- Verify with: `which unzip` and `which zip || python --version`
- If neither zip nor Python is found, guide the user to install per `rules/install.md`

## Workflow

Follow these steps in exact order. Do not skip steps. Read `references/epub-structure.md` before starting if you need a refresher on ePub internals.

### Step 1: Validate Input

1. Confirm the source path exists: `ls -la "<source_path>"`
2. Confirm it has a `.epub` extension
3. If invalid, report the error clearly and STOP

**Directory-form ePub:** A path ending in `.epub` is sometimes a *directory* of already-unpacked ePub content, not a ZIP file. Apple Books libraries, Calibre working folders, and some sync tools store books this way. `unzip` will fail on these with a confusing error, so check first:

```bash
[ -d "<source_path>" ] && echo "DIRECTORY form" || echo "ZIP form"
```

If it is a directory, repack it into a real ePub before continuing, then use the packed file as the source for Steps 2–3:

```bash
src_dir="<source_path>"
packed="<some_scratch_dir>/<book_name>.epub"
cd "$src_dir"
find . -name '.DS_Store' -delete
zip -X -0 -q "$packed" mimetype               # mimetype first, STORED
zip -X -r -9 -q "$packed" . -x mimetype -x '*/.DS_Store'
```

Do NOT write the packed file next to the source — the source directory already occupies that `.epub` name and the write would collide. Use a scratch directory; the final translated ePub goes to `output/` (Step 9.5) regardless.

If the directory has no `mimetype` file, create one containing exactly `application/epub+zip` (no trailing newline) before packing.

### Step 2: Create Work Directory

1. Derive work directory name: `<filename_without_extension>_translation_work/` in the same directory as the source file
2. Create it: `mkdir -p "<work_dir>"`
3. If `_translated/` subdirectory already exists inside it, this is a **resumed session** — report how many checkpoint files exist

### Step 3: Unzip ePub

1. Run: `unzip -o -q "<source_path>" -d "<work_dir>/"`
2. Verify: `ls "<work_dir>/META-INF/container.xml"`
3. If container.xml doesn't exist, report "Invalid ePub: missing META-INF/container.xml" and STOP

**Error checks before proceeding:**
- Check for DRM: `ls "<work_dir>/META-INF/encryption.xml"` — if it exists, read it. If it contains `<EncryptedData>` referencing XHTML files, report "This ePub appears to be DRM-protected and cannot be translated" and STOP.

### Step 4: Locate content.opf

1. Read `<work_dir>/META-INF/container.xml` with the Read tool
2. Find the `full-path` attribute in the `<rootfile>` element
3. This gives you the path to `content.opf` relative to the work directory (e.g., `OEBPS/content.opf`)
4. Read `content.opf` with the Read tool
5. Store the content directory prefix (e.g., `OEBPS/`) — you'll need it for all subsequent file paths

### Step 5: Extract File Manifest

From `content.opf`, extract:

1. **XHTML content files**: All `<item>` elements in `<manifest>` with `media-type="application/xhtml+xml"`. Record their `id` and `href` attributes.
2. **Spine order**: Read `<itemref>` elements in `<spine>`. Map each `idref` to the corresponding manifest item's `href`. This is the reading/translation order.
3. **TOC files**:
   - ePub 2: `<item>` with `media-type="application/x-dtbncx+xml"` → toc.ncx
   - ePub 3: `<item>` with `properties="nav"` → toc.xhtml
4. **Metadata**: Extract `dc:title`, `dc:language`, `dc:description`, `dc:subject`, `dc:creator`
5. **CSS files**: All `<item>` elements with `media-type="text/css"` — record for bilingual CSS injection
6. **Fixed-layout check**: Look for `<meta property="rendition:layout">pre-paginated</meta>` or `<meta name="fixed-layout" content="true"/>`. If found, warn: "This is a fixed-layout ePub. Translation will proceed but layout may be affected."

7. **Text-bearing scan**: Determine which spine files actually contain translatable text. Many books — especially Calibre conversions — render chapter headings as **JPEG images**, leaving the chapter-opener file with no text at all. Run:

   ```bash
   cd "<work_dir>"
   for f in <all spine xhtml files>; do
     python3 -c "
   import re,sys,html
   s=open(sys.argv[1],encoding='utf-8').read()
   b=re.search(r'<body.*?>(.*)</body>',s,re.S)
   t=re.sub(r'<[^>]+>',' ',b.group(1) if b else '')
   t=re.sub(r'\s+',' ',html.unescape(t)).strip()
   print(('TEXT  ' if len(t)>20 else 'NOTEXT'), sys.argv[1], repr(t[:60]))
   " "$f"
   done
   ```

   Record the `NOTEXT` files. They are handled in Step 6.0 without dispatching a subagent.

Report to the user:
- Book title, author, source language
- Number of content files to translate, and how many contain no translatable text
- Whether TOC files were found (ncx, xhtml, or both)
- Whether this is a resumed session (how many already translated)

**If a run of chapter-opener files is `NOTEXT` and each contains an `<img>`**, warn the user explicitly before translating:

> "This book renders its chapter titles as images (N files). Those titles cannot be translated — there is no OCR step. The table of contents and the in-book contents page will still be translated, so navigation will be in the target language, but the chapter-opening pages will show the original-language title image."

Say this upfront rather than in the final report — the user may want to stop and pick a different edition.

### Step 5.5: Establish Translation Style Profile

**Check for existing profile first:** If `<work_dir>/_translated/style_profile.md` exists, read it and use it. Report: "Using existing style profile from previous session." Skip to Step 6.

**If no existing profile**, build one from these sources (in priority order):

1. **User-specified tone** (if provided): Use the user's `--tone` or style hint as the primary directive.

2. **Metadata inference**: From `dc:description`, `dc:subject`, and `dc:title` already extracted in Step 5, analyze:
   - What genre/category is this book?
   - What audience is it for?
   - What tone would be appropriate?

3. **First-chapter style anchor**: Read the first 2-3 paragraphs of meaningful text from the first XHTML content file in spine order (skip cover pages — look for the first file with substantial `<p>` content). Analyze:
   - Vocabulary complexity (technical jargon vs. everyday language)
   - Sentence structure (short and punchy vs. long and complex)
   - Formality level (academic, conversational, casual)
   - Narrative voice (first/second/third person)
   - Use of humor, metaphor, rhetorical devices

**Combine all sources into a Style Profile** and save to `<work_dir>/_translated/style_profile.md`:

```
Genre: [genre]
Tone: [tone description]
Voice: [person and address style]
Vocabulary: [vocabulary approach]
Style notes: [2-3 specific observations about the writing style]
```

This profile will be injected into every translation batch prompt. Show it to the user and ask: "Does this style profile look right? Adjust if needed, or confirm to proceed."

### Step 6: Translate XHTML Files (Parallel Subagents)

Translation uses up to 3 parallel subagents, each handling a subset of files. This ~3x throughput while keeping each subagent's context window manageable.

#### 6.0: Prepare File Assignments

1. Collect all untranslated XHTML files — those in spine order whose checkpoint (`<work_dir>/_translated/<relative_path>`) does NOT yet exist. Print "Skipping (already translated): <filename>" for each skipped file.
2. If **0 files** remain: skip to Step 7.
3. **Pass through the no-text files first.** Using the `NOTEXT` list from Step 5, copy each straight to its checkpoint — never spend a subagent on a file with nothing to translate:
   ```bash
   cd "<work_dir>"
   mkdir -p "_translated/$(dirname "<relative_path>")"
   cp "<relative_path>" "_translated/<relative_path>"
   ```
   Report: "Copied through unchanged (no translatable text): N files". Remove them from the dispatch list. On real books this is often 20–40% of the spine (cover, part dividers, image-only chapter openers), so doing it here can eliminate several whole rounds.
4. **Measure file sizes** of what's left: `wc -c` on each. Classify:
   - **Small**: <30 KB
   - **Medium**: 30–80 KB
   - **Large**: >80 KB
   Report a size summary table to the user (e.g., "3 small, 5 medium, 2 large files").
5. If **1 file** remains: translate it directly in the main agent (use the translation instructions in 6.1.1 below). No subagent overhead needed.
6. If **2+ files** remain: assign files to up to 3 agents using **size-aware bin packing** instead of naive round-robin. The budget per agent is:
   - **Max ~200 KB total** per agent (sum of assigned file sizes) — this is the real constraint; it is what bounds subagent context
   - **Max 8 files** per agent, and **max 3 files above 10 KB** per agent — the file-count limits exist only to keep any single agent from serializing too many substantial translations
   - Assignment algorithm: sort files by size descending, then greedily assign each file to the agent with the smallest current total. This spreads large files across agents rather than clustering them.
   - If only 2 files, use 2 subagents.
   - If files remain after all 3 agents are at capacity, they are handled in the next round (loop back via 6.2).

   **Do not apply the 3-file rule to small files.** A batch of 600-byte chapter-title pages costs an agent almost nothing; capping it at 3 turns a single round into four. Size is the budget, not file count.
7. Read `_translated/style_profile.md` content and `references/translation-prompt.md` content into variables — these will be inlined in each subagent's prompt (see 6.1 for how to avoid re-sending them every round).

#### 6.1: Dispatch Subagents

Launch up to 3 Agent subagents **in a single message** (this is critical — all Agent tool calls must be in one response for true parallel execution).

Each subagent receives a **self-contained prompt** containing:

1. **Role**: "You are a translation subagent. Translate the assigned XHTML files following the instructions below exactly."
2. **Style profile** (inline content, NOT a file path):
   ```
   [STYLE PROFILE]
   <paste full content of style_profile.md>
   [END STYLE PROFILE]
   ```
3. **Translation prompt template** (inline content of `references/translation-prompt.md`)
4. **Work directory path**: `<work_dir>`
5. **Target language**, **source language**, and **output mode** (pure or bilingual)
6. **Assigned files list**: each entry includes the relative path, spine order index, and file size (e.g., "File 3/15: OEBPS/Text/chapter_03.xhtml (85 KB)")
7. **Complete translation instructions** (Steps 6.1.1–6.1.5 below)
8. **Critical strategy directive** (include this verbatim in every subagent prompt):
   ```
   TRANSLATION STRATEGY — READ THIS FIRST:
   For each file, you MUST follow this exact approach:
   1. Read the entire XHTML file using the Read tool
   2. Translate all translatable content in memory
   3. Write the COMPLETE translated file using the Write tool in ONE operation

   Do NOT use the Edit tool for translation. The Edit/find-replace approach is
   too slow for large files, will exhaust your context window, and frequently
   fails when it cannot find unique string matches. The Write-complete-file
   approach is faster, more reliable, and produces consistent results.
   ```

**Multi-round books — write the brief once, don't retype it.** Items 1–3 and 7–8 above are identical for every subagent in every round. On a 50-file book that is 15+ dispatches carrying the same ~2000 words, which burns the orchestrating agent's context for no benefit. Instead, on the **first** round write the shared portion to `<work_dir>/_agent_brief.md` — style profile, translation rules, bilingual format, glossary, per-file procedure, work directory, languages, output mode — and give each subagent a short prompt:

> STEP 0 — MANDATORY FIRST ACTION: Read `<work_dir>/_agent_brief.md` in full and follow it exactly. It contains the style profile, translation rules, bilingual-output format, terminology glossary, and per-file procedure. Do not begin translating before you have read it.
>
> ## Your assigned files
> - File 1/N: `<relative_path>` (size)
> - …

Keep `_agent_brief.md` in the **work directory root**, never inside `_translated/` — anything in `_translated/` gets overlaid into the ePub in Step 9.3.

**Seed the glossary in the brief.** Add a "Terminology to keep consistent" section listing the book's load-bearing terms and their chosen translations. Agents in later rounds never see earlier rounds' output, so this file is the only channel that keeps chapter 20 using the same vocabulary as chapter 1. Extend it between rounds as new key terms surface.

##### 6.1.1: Read the File
- Use the Read tool to read the XHTML file from the work directory
- For files >2000 lines, use offset/limit to read in segments
- Note the complete file structure: XML declaration, `<head>`, `<body>`
- **Encoding check**: If the XML declaration specifies an encoding other than `utf-8` (e.g., `encoding="iso-8859-1"`), warn the user and proceed with caution

##### 6.1.2: Identify Translatable Content
Within `<body>`, identify all translatable block elements:
- `<p>`, `<h1>`–`<h6>`, `<blockquote>`, `<li>`, `<td>`, `<th>`, `<figcaption>`, `<dt>`, `<dd>`

Skip these entirely:
- `<pre>` and `<code>` blocks (source code)
- Empty elements or elements containing only whitespace
- Elements containing only numeric/symbolic content
- Elements containing only URLs

##### 6.1.3: Batch and Translate

Group translatable blocks into batches by **natural semantic boundaries** (sections, heading groups, logical paragraph clusters). Guidelines:
- Aim for ~2000-3000 characters per batch, but this is a guideline, not a hard limit
- **Never split a parent element across batches** (e.g., keep an entire `<blockquote>` or `<table>` together)
- Keep nested structures as single units (e.g., `<ol>` with all its `<li>` children)
- If a single element exceeds 3000 characters, treat it as its own batch

**For each batch:**

1. Use the inlined translation prompt template to remember the translation rules
2. Construct the translation prompt:
   - Insert the inlined style profile into the `{STYLE_PROFILE}` placeholder
   - Set `{TARGET_LANGUAGE}` and `{SOURCE_LANGUAGE}`
   - If this is not the first batch, include the last 2-3 translated paragraphs from the previous batch as context:
     ```
     [PREVIOUS CONTEXT - do not re-translate]
     <p>前一段的翻译内容...</p>
     <p>前二段的翻译内容...</p>
     [END PREVIOUS CONTEXT]

     [TRANSLATE THE FOLLOWING]
     <p>New content to translate...</p>
     ```
3. Translate the batch, producing the target-language HTML
4. For **bilingual mode**: keep the original blocks, and insert translated blocks after each original:
   - Copy the original block element
   - Add `class="translated"` to the translated copy
   - Place the translated copy immediately after the original

##### 6.1.4: Reassemble and Write the Complete XHTML File

Reconstruct the complete XHTML file in memory, then write it in one shot:
1. Keep the XML declaration exactly as-is (e.g., `<?xml version="1.0" encoding="utf-8"?>`)
2. Keep the DOCTYPE exactly as-is (if present)
3. Update `<html>` tag: change `lang="xx"` and `xml:lang="xx"` to the target language code
4. Keep `<head>` section entirely unchanged (title, CSS links, meta tags)
5. Replace `<body>` content with the translated content (or bilingual interleaved content)
6. Create the necessary subdirectories: `mkdir -p "<work_dir>/_translated/<subdirs>/"`
7. **Write the entire file** to `<work_dir>/_translated/<relative_path>` using the **Write tool** — this must be the complete file from XML declaration to closing `</html>` tag, in a single Write call
8. Report: "Translated: <filename> (X/N)"

Each subagent translates its assigned files **sequentially** within its own context, maintaining batch-to-batch "previous context" continuity across files. Steps 6.1.1 through 6.1.4 are repeated for each assigned file.

#### 6.2: Collect and Verify Results

After all subagents complete:

1. **Check checkpoint existence**: `ls <work_dir>/_translated/<relative_path>` for each assigned file. If missing, mark as failed.

2. **Verify translation completeness** for each checkpoint that exists. Verify with Python, not shell text tools:

   ```bash
   cd "<work_dir>"
   python3 - <<'PY'
   import glob, os, re, xml.dom.minidom
   CJK = re.compile(r'[一-鿿぀-ヿ가-힯]')   # widen for your target language
   for p in sorted(glob.glob('_translated/**/*.*html', recursive=True)):
       src = p[len('_translated/'):]
       issues = []
       # a) well-formed XML (this also proves the file is not truncated)
       try: xml.dom.minidom.parse(p)
       except Exception as e: issues.append(f'malformed: {str(e)[:60]}')
       s = open(p, encoding='utf-8').read()
       if not s.rstrip().endswith('</html>'): issues.append('no closing </html>')
       # b) size sanity vs original
       if os.path.exists(src):
           r = os.path.getsize(p) / max(os.path.getsize(src), 1)
           if r < 0.9: issues.append(f'suspiciously small ({r:.2f}x original)')
       print(('FAIL ' if issues else 'ok   ') + os.path.basename(p) + ('  ' + '; '.join(issues) if issues else ''))
   PY
   ```

   **Do not check for `</html>` with `tail -c N | grep`.** Cutting a fixed byte count out of a CJK file lands mid-codepoint; `grep` then treats the input as binary and the check reports failures on perfectly good files. A successful XML parse already proves every tag is closed.

   For **bilingual mode**, also verify pairing — every original block that needs translating should be immediately followed by its translated twin:

   ```python
   blocks = re.findall(r'<(p|h[1-6]|blockquote|li|td|th|figcaption)\b([^>]*)>(.*?)</\1>', body, re.S)
   # for each block WITHOUT 'translated' in its attributes and with >8 latin letters of text,
   # assert the NEXT block HAS 'translated' in its attributes
   ```

   Two things that look like failures but are not, and should not trigger a retry:
   - **Spacer paragraphs, bare URLs, and address lines** legitimately have no translated twin. Filter blocks by "has real prose" before demanding a pair.
   - **Translations containing no target-language characters** — "September 11th" → "9·11", "ARPANET" → "ARPANET". Do not require the twin to match a CJK regex; require only that the twin *exists* and carries the `translated` class.

   If a file is genuinely incomplete, **delete the checkpoint** (`rm`) so it is retried in the next round. Report: "Incomplete translation detected: <filename> — will retry in next round."

3. Report results:
   - "Translated X/N chapters (using K parallel subagents)."
   - If any files were incomplete: "Y files had incomplete translations and will be retried."

**Session management:** If more files remain after this round (including retries from incomplete translations):

- **If the user chose to pause between rounds**, stop and report:
  > "Translated X/N chapters in this round (using K parallel subagents). Checkpoint saved. You can:
  > 1. Continue in this session (will dispatch another round for remaining files)
  > 2. Start a new conversation — just invoke this skill again with the same ePub and I'll pick up where I left off."

- **If the user chose no pausing**, immediately loop back to Step 6.0 to dispatch the next round of subagents for remaining files.

If all files are translated and verified, proceed directly to Step 7 without pausing.

### Step 7: Translate TOC

**Check checkpoint first:** If `_translated/` already has the TOC file(s), skip this step.

**If both `toc.ncx` and `toc.xhtml` exist**, present them to yourself in a single translation batch to guarantee consistent chapter title translations.

**Reuse the in-book contents page.** Most books have a contents/TOC *page* in the spine (a normal XHTML file listing every chapter title) that a subagent already translated in Step 6. Its titles and the navigation labels must match, or the reader gets one wording in the sidebar and another on the page. So:

- When dispatching the subagent that owns the contents page, tell it to append a plain `English → 目标语言` list of every chapter title to its final report.
- Translate `toc.ncx` / `toc.xhtml` here in the main agent using **exactly those titles**.

If no such page exists, translate the navigation labels here and reuse them for any chapter headings still pending.

**Bilingual mode:** translate navigation labels into the target language only — do not interleave both languages in `<navLabel>`. Nav entries are rendered in narrow sidebars where doubled text wraps badly, and the bilingual contents *page* already gives the reader both.

#### toc.ncx (ePub 2)
- Read the file from the work directory
- Translate only the `<text>` content inside each `<navLabel>` element
- Do NOT modify `<content src="..."/>` attributes
- Preserve all XML structure and attributes

#### toc.xhtml (ePub 3)
- Read the file from the work directory
- Translate `<nav epub:type="toc">`: all anchor text in the navigation list
- Translate `<nav epub:type="landmarks">`: translate labels (e.g., "Cover" → target language equivalent)
- Do NOT translate `<nav epub:type="page-list">` entries
- Do NOT modify any `href` attributes
- Update `lang` and `xml:lang` attributes on `<html>` tag

Save both to `_translated/` preserving relative paths.

### Step 8: Update Metadata in content.opf

1. Read the original `content.opf` from the work directory
2. Make these changes:
   - `<dc:language>` → change to target language code (e.g., `en` → `zh`)
   - `<dc:title>` → translate the book title
   - `<dc:description>` → translate the description (if present)
   - `<dc:creator>` → **leave unchanged** (author names are not translated)
3. **Bilingual mode only:** If bilingual CSS needs to be added:
   - If no CSS file was found in Step 5: add a new manifest item for `bilingual.css`
   - This is handled in Step 9
4. Ensure the target directory exists: `mkdir -p "<work_dir>/_translated/<content.opf parent dir>/"`
5. Save the modified content.opf to `<work_dir>/_translated/<content.opf relative path>`

### Step 9: Repackage ePub

#### 9.1: Create Staging Directory
```bash
staging_dir="<work_dir>/_staging"
mkdir -p "$staging_dir"
```

#### 9.2: Copy Original Structure (excluding work artifacts)

Copy only the ePub content directories/files — exclude all work artifacts to prevent non-ePub files from ending up in the final package.

**Derive the copy list from the manifest — do not hardcode `META-INF OEBPS mimetype`.** That triplet is only correct when `content.opf` lives inside `OEBPS/`. When the OPF sits at the ePub root (common in Calibre output), the manifest references root-level files — `content.opf`, `cover.jpeg`, `titlepage.xhtml`, `stylesheet.css`, `toc.ncx` — and a hardcoded copy silently drops every one of them. The result is a ZIP that opens but has no styling, no cover, and no working navigation.

Build the list from what the ePub actually declares:

```bash
cd "<work_dir>"
# Always required
for item in mimetype META-INF; do
  [ -e "$item" ] && cp -R "$item" "$staging_dir/"
done
# Then the OPF itself plus every manifest href, using their top-level path component
#   e.g. hrefs OEBPS/Text/ch1.html, cover.jpeg, stylesheet.css  ->  copy OEBPS, cover.jpeg, stylesheet.css
for item in <opf_path> <unique top-level components of every manifest href>; do
  [ -e "$item" ] && cp -R "$item" "$staging_dir/"
done
```

Copying by top-level component (rather than file by file) preserves the image and font directories the manifest points into.

**Exclude non-manifest files.** Extracted ePubs often carry reader artifacts that are not part of the book: `iTunesMetadata.plist`, `.DS_Store`, `Thumbs.db`, `calibre_bookmarks.txt`, `META-INF/calibre_bookmarks.txt`. Deriving the copy list from the manifest excludes them automatically — one more reason not to `cp -r` whole directories blindly.

#### 9.3: Overlay Translated Files
```bash
# Copy all translated files over the staging copy, preserving directory structure
cp -r "<work_dir>/_translated/"* "$staging_dir/" 2>/dev/null
# Remove non-ePub artifacts that may have been in _translated/
rm -f "$staging_dir/style_profile.md"
```

#### 9.3.1: Verify Staging Directory Cleanliness

Before packaging, verify no artifacts leaked into staging:
```bash
# Check for common artifacts that should NOT be in the ePub
find "$staging_dir" -name "*.py" -o -name "*.txt" -o -name "*.md" -o -name "*.log" \
  -o -name "_*" -type d | head -20
```
If any unexpected files are found, remove them before proceeding.

#### 9.4: CSS Injection

Everything here is **appended** to the existing stylesheet. Never edit or delete a publisher rule — see Prohibitions.

##### 9.4a: Bilingual styles (bilingual mode only)

1. Check if a main CSS file exists in the staging directory (found in Step 5)
2. If yes, append:
   ```css

   /* Bilingual translation styles */
   .translated {
       font-size: 0.95em;
       margin-top: 0.15em;
       padding-left: 0.6em;
       border-left: 2px solid rgba(128, 128, 128, 0.35);
       }
   ```
3. If no CSS file exists:
   - Create `<content_dir>/Styles/bilingual.css` with the above rule
   - Add to `content.opf` manifest: `<item id="bilingual-css" href="Styles/bilingual.css" media-type="text/css"/>`
   - Add a `<link>` reference in each translated XHTML file's `<head>`

**Do not set a hardcoded `color` on `.translated`.** A fixed gray such as `#555555` is legible on white and nearly unreadable in a reader's dark mode, where it becomes dark gray on black. Omitting `color` lets the translated text inherit the reader's theme. The neutral `rgba()` border reads correctly on both light and dark backgrounds and gives the eye a rail for tracking original/translation pairs down a long page.

##### 9.4b: Target-language readability (both modes)

Source stylesheets are written for the source script. A book typeset for English routinely sets no `line-height` at all, or sets ~1.2 — fine for Latin text, cramped to the point of looking broken for CJK, where glyphs are dense and full-height.

If the target language is Chinese, Japanese, or Korean, check the existing CSS for a body-text `line-height`. If none is set, or it is below `1.5`, append a rule scoped to the classes actually used by translated paragraphs:

```css

/* CJK readability for translated text */
.translated, <comma-separated list of body-paragraph classes> {
    line-height: 1.75;
    }
```

Determine the class list from the translated files rather than guessing — in a Calibre conversion it will be names like `.calibre6`, in another book `.para` or nothing at all (in which case target `p`).

Scope and restraint:
- This is **repair, not decoration** — it exists because the translation changed the script, not because the publisher's design was wrong.
- Do NOT add `font-family` (embedding fonts bloats the file and raises licensing questions; readers have good CJK defaults), `text-indent`, drop caps, ornaments, restyled headings, or any other house style. The skill's promise is that the translation looks like the original book.
- If the source CSS already sets an adequate `line-height`, add nothing.
- Mention the appended rules in the final report so the user knows what changed, and honor a request to skip them.

#### 9.5: Package as ePub ZIP

Determine the output path first:

- **Default: `output/<original_name>_<target_lang>.epub`**, where `output/` is in the current working directory. Create it if missing: `mkdir -p output`
- Or the user-specified path, which always wins

Translated books go to `output/` rather than next to the source so that repeated runs collect in one predictable place and never sit adjacent to the originals — which matters when the source is a library directory the user does not want written into. Tell the user the full path in the final report; do not assume they know where `output/` resolved to.

Never write the output *into* the work directory or the source directory tree.

**Option A: If `zip` command is available:**
```bash
cd "$staging_dir"
zip -0 -X "$output_path" mimetype
zip -r "$output_path" * -x mimetype
```

**Option B: If `zip` is not available (common on Windows), use Python:**
```bash
python -c "
import zipfile, os, sys
staging = sys.argv[1]
output = sys.argv[2]
os.chdir(staging)
with zipfile.ZipFile(output, 'w') as zf:
    zf.write('mimetype', 'mimetype', compress_type=zipfile.ZIP_STORED)
    for root, dirs, files in os.walk('.'):
        dirs.sort()
        for f in sorted(files):
            p = os.path.join(root, f)
            arcname = os.path.relpath(p, '.')
            if arcname != 'mimetype':
                zf.write(p, arcname, compress_type=zipfile.ZIP_DEFLATED)
print('ePub packaged successfully')
" "$staging_dir" "$output_path"
```

Check which is available with `which zip` — if not found, use the Python fallback.

#### 9.6: Verify Output

A non-zero file size does not mean the ePub is valid. Validate the package structurally — this is the step that catches a dropped stylesheet or an unresolvable spine before the user opens the book in a reader:

```bash
python3 - <<'PY'
import zipfile, re, xml.dom.minidom
z = zipfile.ZipFile("<output_path>")
names = set(z.namelist())

# 1. mimetype must be the FIRST entry and STORED (uncompressed)
first = z.infolist()[0]
assert first.filename == 'mimetype' and first.compress_type == zipfile.ZIP_STORED, \
       f"mimetype must be first + STORED, got {first.filename}/{first.compress_type}"
assert z.read('mimetype') == b'application/epub+zip'

# 2. container.xml resolves to the OPF
opf_path = re.search(r'full-path="([^"]+)"', z.read('META-INF/container.xml').decode()).group(1)
assert opf_path in names, f"rootfile missing: {opf_path}"
opf = z.read(opf_path).decode()

# 3. every manifest href is present in the zip
hrefs = re.findall(r'<item[^>]*href="([^"]+)"', opf)
missing = [h for h in hrefs if h not in names]
assert not missing, f"manifest files missing from zip: {missing}"

# 4. every spine idref resolves to a manifest item
ids = set(re.findall(r'<item[^>]*id="([^"]+)"', opf))
unresolved = [s for s in re.findall(r'<itemref idref="([^"]+)"', opf) if s not in ids]
assert not unresolved, f"unresolved spine idrefs: {unresolved}"

# 5. every XHTML file still parses after packaging
for h in hrefs:
    if h.endswith(('.html', '.xhtml')):
        xml.dom.minidom.parseString(z.read(h))

assert z.testzip() is None, "zip integrity check failed"
print(f"OK — {len(hrefs)} manifest items, {len(re.findall(r'<itemref', opf))} spine entries, all resolve")
PY
```

If any assertion fails, report the specific failure and STOP — do not hand the user a broken ePub. If the output file is 0 bytes or missing, report a packaging error and STOP.

### Step 10: Cleanup and Report

Report to the user:
- Output file path and size
- Number of chapters translated
- Translation mode used (pure or bilingual)
- Target language

Ask: "Would you like me to keep the work directory (`<work_dir>/`) for debugging, or clean it up?"

If cleanup requested:
```bash
rm -rf "<work_dir>"
```

## Error Handling

| Scenario | How to Detect | Action |
|----------|--------------|--------|
| File not found | `ls` fails | Report "File not found: <path>" and STOP |
| Not .epub | Check extension | Report "Not an ePub file" and STOP |
| Source is a directory | `[ -d "<source_path>" ]` | Repack into a real ePub in a scratch dir (Step 1), then continue |
| Unzip fails | Non-zero exit code | Report "Failed to extract ePub (file may be corrupted)" and STOP |
| Chapter titles are images | Run of spine files with an `<img>` and no text (Step 5 scan) | WARN upfront that those titles cannot be translated, and CONTINUE |
| OPF at ePub root | `container.xml` `full-path` has no `/` | Derive the Step 9.2 copy list from the manifest, not `META-INF OEBPS mimetype` |
| DRM protected | `META-INF/encryption.xml` contains `<EncryptedData>` referencing XHTML files | Report "ePub is DRM-protected" and STOP |
| Fixed-layout | `rendition:layout` = `pre-paginated` in content.opf | WARN "Fixed-layout ePub detected — translation may affect layout" and CONTINUE |
| Non-UTF-8 | XML declaration has `encoding` other than `utf-8` | WARN "Non-UTF-8 encoding detected" and CONTINUE with caution |
| Empty content file | XHTML body has no translatable blocks | Skip silently, write unchanged to checkpoint |
| Super-long paragraph | Single element >3000 chars | Treat as its own batch |
| Translated file exists | `_translated/<path>` already present | Skip (resumability) |
| Output file is 0 bytes | `ls -la` shows 0 size | Report packaging error and STOP |
| zip/unzip not found | `which zip` fails | Direct user to `rules/install.md` and STOP |

## Prohibitions

Do NOT:
- Remove or modify images
- Alter the ePub file/directory structure
- Modify existing CSS rules — only ever append (bilingual styles, and the CJK line-height repair in 9.4b)
- Restyle the book: no embedded fonts, custom font stacks, drop caps, ornaments, recolored or re-sized headings, replacement covers, or any other house style. A translated book should look like the original book, and every rule you add is a rule that can collide with the publisher's design
- Translate content inside `<code>` or `<pre>` tags
- Translate `href`, `src`, `id`, `class`, or other HTML attribute values
- Translate author names
- Add files not required for the translation (no README, no logs inside the ePub)
- Modify spine order or manifest entries (except adding bilingual CSS if needed)

## Limitations

Tell the user upfront:
- Large books (>50 chapters) require multiple conversation sessions (up to 3 parallel subagents per round, checkpoint-based)
- Inline tag preservation is best-effort; complex nesting may occasionally break
- **Text embedded in images is not translated** — and this bites harder than it sounds. Calibre-converted books frequently render every chapter *title* as a JPEG, so those headings stay in the source language even though the body text, contents page, and navigation are all translated. Step 5 detects this and warns before any translation work starts.
- Fixed-layout ePub translations may have layout issues
- SVG text elements are not translated
- `<ruby>`/`<rt>` annotations are preserved as-is
- Table column widths may shift when translated text length differs significantly
- Uses up to 3 parallel subagents for translation — each subagent handles up to ~200 KB, at most 8 files, and at most 3 files over 10 KB
- Large books may require multiple rounds of subagent dispatch; files with no translatable text are copied through in the main agent and never consume a round
- Subagent dispatch has overhead; for books with ≤1 remaining chapters, translation is done directly without subagents
- Resumability checks file existence only; if the source ePub changes between runs, delete `_translated/` to force re-translation
- Bilingual mode appends a minimal `.translated` rule (slightly smaller text, a neutral left rule); it deliberately sets no `color` so the text follows the reader's light/dark theme. Users may customize further
- For CJK targets the skill may append a `line-height` rule when the source stylesheet has none — repairing leading that was set for Latin text. It makes no other styling changes: no fonts, no ornaments, no restyled headings
