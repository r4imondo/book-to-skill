---
name: book-to-skill
description: "Converts books and documents (PDF, EPUB, DOCX, HTML, Markdown, plain text, RTF, MOBI/AZW with Calibre) into structured agent skills, extracting frameworks, mental models, principles, techniques, and anti-patterns. Use when the user wants to study a document through Amp or Claude Code, apply an author's frameworks while working, or build a reusable knowledge base from a file."
compatibility: "Amp skill directories (.agents/skills, ~/.config/agents/skills, ~/.config/amp/skills) and Claude Code skill directories (~/.claude/skills)."
allowed-tools:
  - shell_command
  - Read
  - Write
  - Glob
  - Grep
argument-hint: <path-to-document> [skill-name-slug]
---

# Book-to-Skill Converter

Transform written knowledge into actionable agent skills by extracting structure — not producing summaries.

## Philosophy

Books contain crystallized expertise: frameworks, principles, and techniques that took years to develop. This skill extracts that knowledge into a format Amp, Claude Code, or another compatible agent can leverage repeatedly.

**Extract structure, not summaries.** A skill isn't a book report. It's a toolkit of:
- Named frameworks (mental models with clear application)
- Actionable principles (rules that guide decisions)
- Techniques (step-by-step methods)
- Anti-patterns (what to avoid and why)
- Voice calibration (how the author thinks and communicates)

**Preserve the author's precision.** Frameworks often have specific names for reasons. "The 5 Whys" isn't interchangeable with "ask why multiple times." Capture the exact formulation.

**Layer depth appropriately.** Simple books → simple skills. Complex books with 10+ frameworks → skills with reference files and on-demand chapters.

---

## Modes of Operation

Three paths available. Route based on what the user asks:

### 1. Full Conversion (Default)
**Trigger:** User provides a supported document path without special instructions
**Action:** Run all steps below (Steps 0–9)
**Output:** Complete skill with SKILL.md, chapters/, glossary, patterns, cheatsheet

### 2. Analyze Only
**Trigger:** User says "analyze", "just extract", or "I want to review before generating"
**Action:** Run Steps 0–3, then produce a structured extraction report (frameworks, principles, techniques found). Stop — do NOT generate skill files.
**Output:** Analysis report for user review

### 3. Generate from Prior Analysis
**Trigger:** User has existing analysis notes or previously ran analyze-only
**Action:** Skip Steps 0–3, use the provided analysis as input, run Steps 4–9
**Output:** Skill files from the provided analysis

---

## Skill Locations

This converter can run from multiple skill systems. When looking for this converter's helper script or writing the generated book skill, prefer these locations in order:

1. Amp project-local skills: `.agents/skills/`
2. Amp global skills: `~/.config/agents/skills/`
3. Amp legacy global skills: `~/.config/amp/skills/`
4. Claude Code skills: `~/.claude/skills/`

Generated skills should default to `~/.config/agents/skills/` for Amp unless the user asks for project-local or Claude Code output.

---

## Step 0 — Out-of-scope check

If the argument is NOT a path to a supported document file, stop and respond:
> "book-to-skill requires a supported document path. Usage: `book-to-skill /path/to/book.pdf [skill-name]`, `book-to-skill /path/to/book.epub [skill-name]`, or another supported format such as `.docx`, `.md`, `.txt`, `.html`, `.rtf`, `.mobi`, or `.azw3`."

Throughout the workflow, treat the first argument as `BOOK_PATH` and the optional second argument as `SKILL_NAME`.

---

## Step 1 — Validate input

```bash
test -f "$BOOK_PATH" && echo "FILE_OK" || echo "FILE_NOT_FOUND: $BOOK_PATH"
case "${BOOK_PATH##*.}" in
  pdf|PDF|epub|EPUB|docx|DOCX|txt|TXT|md|MD|markdown|MARKDOWN|rst|RST|adoc|ADOC|asciidoc|ASCIIDOC|html|HTML|htm|HTM|rtf|RTF|mobi|MOBI|azw|AZW|azw3|AZW3) echo "FORMAT_OK" ;;
  *) echo "FORMAT_UNKNOWN" ;;
esac
```

Check the file extension (`.pdf`, `.epub`, `.docx`, `.txt`, `.md`, `.markdown`, `.rst`, `.adoc`, `.html`, `.htm`, `.rtf`, `.mobi`, `.azw`, `.azw3`) or magic bytes (`%PDF` or `PK` zip header for EPUB/DOCX).

If the file is not found or the format is not supported, stop with a clear error message listing supported formats.

---

## Step 1.5 — Identify book type

Before extracting, ask the user:

> "What kind of content does this book have? This helps me choose the best extraction method.
>
> 1. **Technical** — has code blocks, tables, formulas, diagrams (e.g. programming books, academic papers, architecture guides)
> 2. **Text-heavy** — mostly prose, few or no tables/code (e.g. management, productivity, narrative non-fiction)
> 3. **Not sure** — I'll use the fast method and warn you if quality seems limited"

Store the answer as `BOOK_TYPE`:
- Option 1 → `BOOK_TYPE=technical`
- Option 2 → `BOOK_TYPE=text`
- Option 3 → `BOOK_TYPE=text`

**If `BOOK_TYPE=technical`**, inform the user before proceeding:
> "📐 Technical mode selected — using Docling for structure-aware extraction (tables, code blocks, formulas preserved as markdown). This takes ~1.5s per page, so expect a few minutes for longer books. Starting now…"

**If `BOOK_TYPE=text`**, inform:
> "📄 Text mode selected — using the fastest suitable extractor for this file type. Plain text/Markdown/HTML are usually ready in seconds; PDFs use pdftotext when available."

---

## Step 2 — Extract text from the source document

Run the extraction script, passing the book type:

```bash
SCRIPT_PATH=""
for candidate in \
  ".agents/skills/book-to-skill/scripts/extract.py" \
  "$HOME/.config/agents/skills/book-to-skill/scripts/extract.py" \
  "$HOME/.config/amp/skills/book-to-skill/scripts/extract.py" \
  "$HOME/.claude/skills/book-to-skill/scripts/extract.py"
do
  if [ -f "$candidate" ]; then
    SCRIPT_PATH="$candidate"
    break
  fi
done

if [ -z "$SCRIPT_PATH" ]; then
  echo "Could not find scripts/extract.py for book-to-skill" >&2
  exit 1
fi

PYTHON_BIN="${PYTHON_BIN:-/Users/aria/Agency/tools/book-to-skill/.venv/bin/python}"
if ! command -v "$PYTHON_BIN" >/dev/null 2>&1; then
  PYTHON_BIN="python"
fi

"$PYTHON_BIN" "$SCRIPT_PATH" "$BOOK_PATH" --mode <BOOK_TYPE> --install-missing ask
```

Before extraction, the script checks optional Python packages needed for the detected format. If a better extractor is missing, it prompts the user with the available fallback, for example:

> "DOCX extraction uses python-docx if installed, otherwise a stdlib ZIP/XML parser. Missing package(s) detected. Do you want to install? y=install, n=fallback"

Use `--install-missing yes` to install missing Python packages without prompting, `--install-missing no` or `--no-install-missing` to always use fallbacks, or `BOOK_SKILL_INSTALL_MISSING=yes|no|ask` to set the behavior by environment variable. Non-interactive sessions default to fallback unless install mode is explicitly `yes`.

- PDF `--mode technical` → uses Docling (layout-aware, preserves tables and code blocks as markdown)
- PDF `--mode text` → uses pdftotext → PyPDF2 → pdfminer fallback chain (fast, plain text)
- EPUB → uses ebooklib + BeautifulSoup4, then stdlib ZIP/HTML fallback
- DOCX → uses python-docx, then stdlib ZIP/XML fallback
- TXT/Markdown/reStructuredText/AsciiDoc → reads directly as text
- HTML → uses BeautifulSoup4, then stdlib HTML fallback
- RTF → uses striprtf, then a basic regex fallback
- MOBI/AZW/AZW3 → uses Calibre `ebook-convert` when installed. Calibre is an external app, not a pip package, so the script reports how to install it if missing.

This creates:
- `<tempdir>/book_skill_work/full_text.txt` — full extracted text
- `<tempdir>/book_skill_work/metadata.json` — title, estimated pages, token count, size, extraction_mode

Read the `output_text` path in `<tempdir>/book_skill_work/metadata.json` to understand what was extracted. The extractor uses the platform temp directory by default and supports `BOOK_SKILL_WORKDIR` if an explicit work directory is needed.

---

## Step 2.5 — Pre-flight cost estimate

Read `<tempdir>/book_skill_work/metadata.json` and present the user with an estimate **before doing any generation**:

```
📖 Source detected: <filename> (<format>)
📄 Pages/Spine items/Sections: ~<N> | Words: ~<N> | Source tokens: ~<N>K

💰 Estimated token cost (Full Conversion):
   Input  (book reading + prompts): ~<N>K tokens
   Output (skill files generated):  ~<N>K tokens
   Total:                           ~<N>K tokens

   Reference prices (as of 2025):
   Claude Sonnet 4.5 → ~$<X> USD
   Claude Haiku 4.5  → ~$<X> USD

   ⏱  Estimated time: ~<N> minutes

📁 Files to be generated:
   SKILL.md + <N> chapter files + glossary + patterns + cheatsheet

➡  Proceed with Full Conversion? (or type "analyze only" to preview first)
```

**How to estimate:**
- Input tokens ≈ `estimated_tokens` from metadata × 1.3 (prompts overhead per chapter pass)
- Output tokens ≈ chapters × 1,000 + 4,000 (SKILL.md) + 4,500 (glossary + patterns + cheatsheet)
- Price: Sonnet input=$3/MTok output=$15/MTok — Haiku input=$0.80/MTok output=$4/MTok

Wait for the user to confirm before proceeding. If they say "analyze only", switch to Mode 2.

---

## Step 2.6 — REPL-style access for large books (> 50k tokens)

Inspired by the Recursive Language Model (RLM) paradigm: treat `full_text.txt` as a queryable corpus, not a single read. Loading the whole file into context burns budget you will need later for generation.

For books over ~50k tokens, prefer programmatic probes over `Read(full_text.txt)` without bounds:

```bash
# Size check before any Read
wc -w "$FULL_TEXT_PATH"

# Find chapter offsets without loading the whole file
grep -n -E "^\s*(Chapter|CHAPTER)\s+[0-9]+" "$FULL_TEXT_PATH" | head -40

# Pull only the chapter you need (lines start..end inclusive)
sed -n '<start>,<end>p' "$FULL_TEXT_PATH"

# Verify a framework is actually mentioned before claiming it in SKILL.md
grep -c -i "westrum\|dora" "$FULL_TEXT_PATH"

# Targeted Read with offset/limit avoids dumping the full file
# Read(file_path=full_text.txt, offset=<line>, limit=<lines>)
```

Use this approach for Step 3 (structure analysis), Step 7 (per-chapter summaries), and Step 8 (glossary / patterns extraction). On books under 50k tokens, a single `Read` is fine.

Why this matters: a 200-page book is ~75k tokens. Re-reading it once per chapter (28 passes) costs ~2M input tokens; using grep + sed to pull only relevant slices keeps generation cost proportional to the output, not the source.

---

## Step 3 — Analyze book structure

Read the first 8,000 characters of the extracted `full_text.txt` to identify:
- Book **title** and **author(s)**
- **Chapter structure** (look for "Chapter N", "PART I", numbered headings, table of contents)
- **Core themes** and subject domain
- Approximate number of chapters

Then read the Table of Contents section if present to map all chapters.

**If mode is "Analyze Only":** produce the extraction report now and stop. Structure:
```
## Extraction Report — <Title>

### Author's Core Frameworks
- **<Framework Name>**: <what it is and when to apply>

### Key Principles
- <Principle>: <actionable rule>

### Techniques & Methods
- <Technique>: <step-by-step or how-to>

### Anti-patterns
- <What to avoid>: <why>

### Suggested Skill Name
`{author-lastname}-{core-concept}` — e.g. `cialdini-influence`

### Chapters Detected
| # | Title | Main Frameworks |
```

---

## Step 4 — Ask purpose (Full Conversion only)

Before generating, ask the user:

> "What should this skill help you do? (Pick one or more)
> 1. Apply the author's frameworks while working
> 2. Think with the author's mental models
> 3. Reference specific chapters and concepts
> 4. All of the above"

Use the answer to weight what gets highlighted in the SKILL.md Core section.

---

## Step 5 — Determine skill name

If `SKILL_NAME` was provided, use it as the skill slug.
Otherwise, propose two options and let the user choose:
- **By author-concept**: `{author-lastname}-{core-concept}` (e.g. `cialdini-influence`, `meadows-systems`)
- **By title**: lowercase hyphens from book title (e.g. `designing-data-intensive-apps`)

Default to author-concept format if the book has a strong methodological identity.

Choose the destination skill root:
- **Amp default**: `~/.config/agents/skills`
- **Amp project-local**: `.agents/skills` when the user explicitly wants the generated book skill scoped to the current workspace
- **Amp legacy**: `~/.config/amp/skills` if that is the user's existing global skill location
- **Claude Code**: `~/.claude/skills` if the user explicitly asks for Claude Code output

Set `SKILLS_HOME` to the selected root and check that `$SKILLS_HOME/<skill_name>/` does NOT already exist.
If it does, append `-2` or ask the user before overwriting.

---

## Step 6 — Create skill directory structure

```bash
mkdir -p "$SKILLS_HOME/<skill_name>/chapters"
```

---

## Step 6.5 — Grounding Discipline (CRITICAL)

Before generating any chapter summary, glossary entry, pattern, or master SKILL.md content,
internalize this discipline. The synthesis steps (7, 8, 9) MUST respect it.

### The core rule

**Every framework, principle, technique, anti-pattern, number, and named tool in the
generated skill MUST be traceable to the source `full_text.txt`.** If you cannot point
to the line(s) in `full_text.txt` that support a claim, the claim does not belong in
the skill — even if you "know" it from training and it is factually correct in the
broader domain.

This rule exists because the model performing the synthesis has strong priors on the
book's domain (marketing, software, productivity, etc.) and will, by default, complete
the book with domain knowledge it has from training. That completion is hallucination
from the skill's perspective: the user will read it and believe "the book says X" when
the book does not.

### Faithful inference (REQUIRED) vs Training completion (FORBIDDEN)

These two are different and the distinction is critical. Do not collapse into timidity
that loses faithful inference.

- **Faithful inference (DO)**: the book describes a concept without naming it, or names
  it once and you recognize the concept in another passage. Capture it consolidated.
  Example: the book describes step 1, 2, 3 of an approach without giving it a label —
  you may state the approach as a coherent named framework, attributing it to the book.
- **Training completion (DO NOT)**: the book mentions topic X; you "complete" it with
  related-but-not-stated content from your training. The completion may be factually
  correct, but the book did not say it.

### Specific failure modes to avoid (from real audit findings)

**Your task** is to faithfully extract the book's structure at the book's own level of
abstraction. Recognizing a concept the book expresses in different words is **faithful
inference and IS required**. Adding external knowledge the book does not contain is
**completion and is forbidden**. When in doubt between the two, **prefer the book's
formulation**.

The patterns below are concrete failure modes observed when synthesis models convert
books in specialist domains. Each is forbidden unless the book contains it. Read them
as a sharpening of judgment, not as a license for timidity.

1. **Product/feature terminology not in the text.** If the book discusses a problem
   without naming the platform feature that solves it, do NOT introduce the feature
   name as if the book had named it. (Real example from a Google Ads book audit:
   model added "Brand exclusions PMax" as a framework, but the book only referenced
   the underlying problem of "Search brand cannibalization" without ever naming the
   feature.) Common categories:
   - Vendor feature names (Brand exclusions, Enhanced Conversions, Maximize
     Conversions, Expanded Text Ad, Local campaigns, Free listings, etc.)
   - Web/design jargon not in the text (above-the-fold, hero section, etc.)
   - Frameworks of the trade not named by the author (AIDA, PAS, BANT, MEDDIC, 4 Ps,
     Cialdini principles, StoryBrand, etc.)

2. **Numbers, percentages, thresholds, durations, cadences not in the text.** If the
   book says "scale gradually", do NOT add "+10-20% per step" unless the book gives
   the percentage. If the book says "ensure enough conversions", do NOT add ">30
   minimum per variant" or ">100 better". If the book says "wait several weeks
   before judging", do NOT add "2-3 weeks of good data" or "1-2 weeks between
   steps". These thresholds and timings may be standard industry advice but they
   are not the book's. (Real examples: "70% feed / 30% rest", ">50% mobile
   traffic", "30/100 conv per variant test", "2-6 weeks setup", "1-3 hours
   audit/month", "2-3 weeks of good data before scaling", "1-2 weeks observation
   between scaling steps", "1-2 hours per client for monthly report", "30 minutes
   initial education conversation" — all plausible, none in the book.)

3. **Anti-patterns from your training, presented as the book's.** Common-sense
   marketing or engineering errors are NOT automatically the book's anti-patterns.
   Only include anti-patterns the book itself frames as things to avoid. (Real
   examples added by mistake: "click fraud on Display", "landing without social
   proof in B2B high-ticket", "CTA hidden below the fold", "long form with
   technical fields at first-conversation stage" — all reasonable, none stated as
   anti-patterns in the book.)

4. **Didactic metaphors and analogies you invent.** Even if your metaphor clarifies
   the book's concept, presenting it inside the skill makes the user think "this
   metaphor is in the book". Either omit metaphors of your own creation, or label
   them explicitly as your gloss (see labelling convention below). (Real examples
   to avoid: "Broad + Smart Bidding = telescope + autopilot", "RSA as LEGO blocks",
   "Audience signals = Bayesian prior", "Funnel as conditional probabilities".)

5. **Specific vendor names as examples** when the book is generic. If the book says
   "use a CRM" without naming brands, do NOT add "(e.g. HubSpot, Salesforce,
   Pipedrive)". (Real examples added by mistake: Zapier, ActiveCampaign, plus LLM
   vendor names like GPT/Claude/Gemini when the book only said "language models".)
   Only name vendors the book itself names.

6. **Expansion of a prose concept into an official formula or structured notation.**
   If the book describes a mechanic in qualitative prose (e.g. "Ad Rank is
   calculated based on bid, ad quality, and landing experience"), do NOT rewrite
   it as the product's official formula (e.g. "Ad Rank = bid × Quality Score
   (expected CTR + ad relevance + landing experience) + ext impact + context").
   The formula may be correct in the broader product documentation, but the book
   did not state it that way. Keep the book's level of abstraction.

7. **Standard industry taxonomies imposed over the book's own categorization.** If
   the book proposes its own tripartition / list / segmentation, do NOT overwrite
   or "enrich" it with the standard textbook taxonomy from your training. Examples
   of this failure mode:
   - The book uses "capture demand / create demand / reactivation" → do NOT
     restructure into "Awareness / Consideration / Conversion / Retention" funnel.
   - The book lists three diagnostic KPIs (impression, click, CTR) → do NOT expand
     to "Search impression share, Quality Score, Position, Bounce rate, LTV/CAC".
   - The book defines five funnel stages → do NOT remap them onto AARRR or any
     other model.
   The book's categorization may be less canonical than the textbook version, but
   the user is studying the book, not the textbook. Preserve the book's structure.

8. **Qualitative directive expanded into formal ordering, rationale, named
   consequence, or taxonomy.** When the book provides a qualitative directive —
   extremes without ordering, a principle without rationale, a prescription
   without a named technical consequence — do NOT add formal ordering, rationale,
   named consequence, or taxonomy unless textually present in the book. The model
   tends to "fill" both explanatory space (in descriptive/strategic books) and
   operational space (in prescriptive/tactical books) — both are the same failure
   mode in different forms. Real examples from two audits on books in different
   languages and editorial styles:
   - **Taxonomy expansion** (descriptive book): the book stated only "native
     smart bidding sits at the lower end of bidding maturity, while propensity
     modelling would sit higher" — two extremes, no ordering. The synthesis
     built a 4-rung ordered scale ("Lower / Mid / Upper-mid / Higher end") as if
     the book had defined four levels.
   - **Action expansion** (prescriptive book): the book said "monitor schedules
     and geographies regularly to cut waste". The synthesis added "reduce time
     slots with low CVR / reduce regions with CPA above target" as if these
     specific actions were the book's prescription — they are consultant
     practice, not the book.
   - **Rationale expansion**: the book said "Avoid minimum CPCs". The synthesis
     added "(interferes with learning)" as the inferred reason — the reason was
     never stated. Same pattern with "starves system liquidity" added as the
     reason for "avoid over-using negatives".
   When the book leaves explanatory or operational space (an extreme without a
   middle, a directive without a rationale, a principle without a named
   consequence), do NOT fill it from training. Keep the book's level of
   specificity, or label the addition with the convention below.

### The labelling convention for justified non-grounded additions

Sometimes — after applying the discipline above — you still judge that a non-grounded
addition is useful to the reader (a well-known external pointer, a standard formula
the book references only qualitatively, a clarifying example the book omits). For
these cases there is an explicit labelling convention.

**An addition correctly labelled `[non dal libro]` is the right behavior, not a
failure.** It is transparency: the reader sees both the book's content and the bridge
to external knowledge, and knows which is which. The failure mode is the **silent
addition** — the unlabelled completion that the reader will mistake for the book's
content. An audit reviewer who treats a labelled addition the same as a silent
addition is itself reviewing incorrectly.

How to label:

- Use the marker **`[non dal libro]`** (Italian) or **`[not from the book]`** (English)
  — match the skill's language.
- The label must be on the same paragraph as the addition, not buried in a footnote.
- A short reason or pointer is preferred: `[non dal libro — feature standard di
  Google Ads]` is better than `[non dal libro]` alone.

Examples of correct labelled additions:

- *"Cannibalizzazione Search brand: il libro identifica il problema; la soluzione
  operativa più comune in Google Ads è la feature 'brand exclusions' [non dal libro
  — termine non usato esplicitamente nel testo]."*
- *"Use a CRM (HubSpot, Salesforce, etc. [non dal libro — esempi di vendor; il
  testo non nomina nessuno specifico])."*

### When in doubt, write the absence

If a chapter clearly refers to a topic but the book does NOT supply the operational
content (scripts, templates, exact thresholds, case studies), do NOT fill the gap
from training. Instead, write a brief "Note: parts the book does not detail" section
at the bottom of that chapter file, listing what the book references without
supplying. This is honest, lets the user know where to look elsewhere, and avoids
the temptation to complete from training.

### Future enhancement candidate

A standalone `book-to-skill-audit` skill is a planned evolution: it would automate
the cold-start grounding verification (read full_text + generated files; report
findings by severity; check labelling convention compliance) as a standardized
post-process step. For now this verification is done manually by spawning an
independent agent with explicit grounding criteria. The standalone skill would make
the audit step repeatable and consistent across conversions, valuable for commercial
use of book-to-skill.

---

## Step 7 — Generate chapter summaries

**TOKEN BUDGET RULE — CRITICAL:**
- Each chapter summary file: **800–1,200 tokens** (dense, not verbose)
- Files are loaded on-demand — they are NOT capped per se, but keep them useful and tight

For EACH chapter/major section identified in Step 3:

Read the corresponding section of the extracted `full_text.txt` (use character offsets or grep for chapter headings).

Create `$SKILLS_HOME/<skill_name>/chapters/ch<NN>-<slug>.md` using the structure below.

**Adapt emphasis based on `BOOK_TYPE`:**
- `technical` → prioritize "Code Examples", "Reference Tables", and "Commands & APIs" sections; preserve exact syntax
- `text` → prioritize "Frameworks Introduced", "Mental Models", and "Key Takeaways"; skip empty technical sections

```markdown
# Chapter N: <Full Title>

## Core Idea
<1–2 sentences: the single most important thing this chapter teaches>

## Frameworks Introduced
- **<Framework Name>**: <exact formulation — preserve the author's naming>
  - When to use: <specific situation>
  - How: <steps or criteria>

## Key Concepts
- **<Term>**: <precise definition in 1 sentence>
(5–10 most important terms from this chapter)

## Mental Models
<2–4 frameworks or thinking tools. Write as "Use X when Y" or "Think of X as Y">

## Anti-patterns
- **<What to avoid>**: <why it fails>

## Code Examples *(technical books only — omit if BOOK_TYPE=text)*
<!-- Copy the most instructive snippet from the chapter. Preserve indentation exactly. -->
```<language>
<key code example from this chapter>
```
- **What it demonstrates**: <one line>

## Reference Tables *(technical books only — omit if BOOK_TYPE=text)*
<!-- Reproduce any comparison matrix, parameter table, or decision table from the chapter in markdown. -->

## Key Takeaways
1. <Actionable insight>
2. <Actionable insight>
3. <Actionable insight>
(3–7 takeaways a practitioner must remember)

## Connects To
- **Ch N**: <why this chapter relates>
- **<Concept>**: <external concept or standard it connects with>
```

---

## Step 8 — Generate supporting files

### glossary.md
Create `$SKILLS_HOME/<skill_name>/glossary.md`:
- Every significant term from the book, alphabetically sorted
- Format: `**Term** — definition (Ch N)`
- Max 1,500 tokens

### patterns.md
Create `$SKILLS_HOME/<skill_name>/patterns.md`:
- All concrete techniques, design patterns, algorithms from the book
- Format: `## Pattern Name\n**When to use**: ...\n**How**: ...\n**Trade-offs**: ...`
- Max 2,000 tokens

### cheatsheet.md
Create `$SKILLS_HOME/<skill_name>/cheatsheet.md`:
- Decision tables, comparison matrices, quick-reference rules
- The content you'd want on a single printed page
- Max 1,000 tokens

---

## Step 9 — Generate the master SKILL.md

**CRITICAL TOKEN BUDGET: Keep SKILL.md body under 4,000 tokens.**
Compaction truncates from the END — put the most important content FIRST.

Create `$SKILLS_HOME/<skill_name>/SKILL.md`:

```markdown
---
name: <skill_name>
description: "Knowledge base from \"<Full Title>\" by <Author(s)>. Use when applying <author>'s frameworks for <key topics, 3–6 terms>, studying the book, or referencing its concepts."
allowed-tools:
  - Read
  - Grep
argument-hint: [topic, framework name, or chapter number]
---

# <Full Title>
**Author**: <Author(s)> | **Pages**: ~<N> | **Chapters**: <N> | **Generated**: <YYYY-MM-DD>

## How to Use This Skill

- **Without arguments** — load core frameworks for reference
- **With a topic** — ask about `replication`, `pricing`, or another indexed topic; I find and read the relevant chapter
- **With chapter** — ask for `ch05`; I load that specific chapter
- **Browse** — ask "what chapters do you have?" to see the full index

When you ask about a topic not covered in Core Frameworks below, I will read
the relevant chapter file before answering.

---

## Core Frameworks & Mental Models
<!-- ~2,000 tokens: the author's most important named frameworks and principles.
     Preserve exact names. Write as "Use X when Y", "Prefer X over Y because Z".
     This is a toolkit, not a summary. -->

<generate 2,000 tokens of the most critical frameworks and insights here>

---

## Chapter Index

| # | Title | Key Frameworks |
|---|-------|----------------|
| [ch01](chapters/ch01-<slug>.md) | <Title> | <framework1>, <framework2> |
| [ch02](chapters/ch02-<slug>.md) | <Title> | <framework1>, <framework2> |
...

## Topic Index

<!-- Alphabetical. Major terms/frameworks → chapter(s) that cover them. -->
- **<Term>** → ch<N>[, ch<N>]
- **<Term>** → ch<N>

## Supporting Files

- [glossary.md](glossary.md) — all key terms with definitions
- [patterns.md](patterns.md) — all techniques and design patterns
- [cheatsheet.md](cheatsheet.md) — quick reference tables and decision guides

---

## Scope & Limits

This skill covers the book content only. For hands-on implementation in your codebase,
combine with project-specific tools. For topics beyond this book, check related skills
or ask the agent directly.
```

---

## Step 10 — Cleanup and report

```bash
PYTHON_BIN="${PYTHON_BIN:-/Users/aria/Agency/tools/book-to-skill/.venv/bin/python}"
if ! command -v "$PYTHON_BIN" >/dev/null 2>&1; then
  PYTHON_BIN="python"
fi

"$PYTHON_BIN" - <<'PY'
import os
import shutil
import tempfile
from pathlib import Path
shutil.rmtree(
    os.environ.get("BOOK_SKILL_WORKDIR", Path(tempfile.gettempdir()) / "book_skill_work"),
    ignore_errors=True,
)
PY
```

Then report to the user:

```
✅ Skill created: $SKILLS_HOME/<skill_name>/

📚 Book: <Full Title> — <Author>
📄 Pages: ~<N> | Chapters: <N>

Files generated:
  SKILL.md         — core frameworks + index   (~X tokens)
  chapters/        — <N> chapter summaries     (~X tokens each, ~X total)
  glossary.md      — key terms                 (~X tokens)
  patterns.md      — techniques & patterns     (~X tokens)
  cheatsheet.md    — quick reference           (~X tokens)
  ─────────────────────────────────────────────────────
  Total skill size: ~X tokens (loaded on-demand, not all at once)

💡 Tip: check your agent's session cost/usage command to see actual token usage.

Usage:
  Ask for <skill_name>                  → load core frameworks
  Ask <skill_name> about <topic>        → find and explain a topic
  Ask <skill_name> for ch<N>            → dive into a specific chapter
```

---

## Quality Rules

1. **Extract structure, not summaries** — capture named frameworks, exact formulations, anti-patterns; not chapter recaps
2. **Preserve the author's precision** — "The 5 Whys" ≠ "ask why multiple times"; keep exact naming
3. **Density over completeness** — a 1,000-token summary beats a 10,000-token excerpt
4. **Practitioner voice** — write "Use X when Y", not "The book explains X"
5. **Front-load SKILL.md** — compaction keeps the first 5,000 tokens; most important content comes first
6. **Chapter files are on-demand** — they don't count against skill budget until loaded
7. **Never copy raw book text** — always synthesize, summarize, extract signal
8. **Topic index is critical** — it's how the agent navigates to the right chapter file
9. **Ground every claim** — every framework, principle, technique, anti-pattern, named
   tool, number, threshold, duration, and cadence must be traceable to the source text.
   See Step 6.5 for the discipline and the labelling convention for justified
   non-grounded additions.
10. **Label non-grounded additions inline** — use `[non dal libro]` / `[not from the
    book]` markers when an addition cannot be grounded. Better an explicit label than
    silent completion that the reader will mistake for the book's content. A correctly
    labelled addition is transparency, not failure; the failure is the silent addition.
