# PDF → Markdown: Model × Harness Benchmark

A natural experiment in how model capability and agentic harness design interact. The same prompt — *"Convert the two PDFs to Markdown including some YAML frontmatter describing their source"* — was issued to three models through three AI coding harnesses, producing a 3 × 3 matrix of outputs.

**Source documents:** two browser-printed PDF articles about the MASQUE tunneling protocol (10 and 16 pages). Both carry repeating page headers, footer chrome, and page-number stamps. The InstaTunnel article additionally uses an OpenType ligature font that some extraction tools fail to decode.

A full annotated comparison with code excerpts is in [`benchmark.html`](benchmark.html) (open locally).

---

## Grade Summary

| | Claude Code | OpenCode | Copilot |
|---|:---:|:---:|:---:|
| **Sonnet 4.6** | A | A | B |
| **Kimi K2.6** | A− | C+ | F |
| **DeepSeek-V4-Pro** | B+ | F | F |

---

## Detailed Results

| Model | Harness | PDF method | Ligatures | Artefacts stripped | Frontmatter | Code blocks | Headings | Grade |
|---|---|---|:---:|:---:|---|:---:|:---:|:---:|
| **Sonnet 4.6** | Claude Code | Images (Read tool) | ✓ | ✓ | Rich — title, author, date, URL, tags | ✓ | ✓ | **A** |
| **Sonnet 4.6** | OpenCode | Images + extracted text | ✓ | ✓ | Rich + `pdf_file` field | ✓ | ✓ | **A** |
| **Sonnet 4.6** | Copilot | Custom PyMuPDF pipeline (8 iter.) | ✓ | ⚠ Partial | Rich — source URL, author, topics | ✗⁴ | ✓ | **B** |
| **Kimi K2.6** | Claude Code | pdfplumber | ✓ | ✓ | Rich — title, author, date, URL | ✓ | ✓ | **A−** |
| **Kimi K2.6** | OpenCode | pymupdf4llm | ⚠ Partial¹ | ✗ | File metadata only | ✗ | ⚠ Near-miss² | **C+** |
| **Kimi K2.6** | Copilot | Instant raw dump | ✗ | ✗ | Empty title/author; hallucinated URL⁵ | ✗ | ✓ | **F** |
| **DeepSeek-V4-Pro** | Claude Code | PyMuPDF (fitz) | ✓ | ✓ | Rich + PDF creator/producer metadata | ✓ | ⚠ Partial³ | **B+** |
| **DeepSeek-V4-Pro** | OpenCode | Raw binary extraction | ✗ | ✗ | Empty fields | ✗ | ✓ | **F** |
| **DeepSeek-V4-Pro** | Copilot | Instant raw dump | ✗ | ✗ | Filename as title, no source URL | ✗ | ✓ | **F** |

¹ pymupdf4llm decoded ligatures in the Cloudflare PDF but failed on the InstaTunnel PDF's font encoding.  
² One false heading: `## it intentionally.` — a page-break sentence fragment misidentified as a heading.  
³ `The current state of WireGuard` incorrectly nested as H3; one paragraph detached from its section heading.  
⁴ The Extended CONNECT request code block was lost in extraction; page chrome partially retained.  
⁵ Only fabrication in the entire benchmark: `source_url: "https://blog.unnic.com"` — invented, not present in source.

---

## Key Findings

### 1. Harness behavioral shaping is a first-order variable

The Kimi K2.6 comparison is the clearest evidence: the same model ranged from **C+ to A−** depending solely on which harness it ran through. The Claude Code system prompt encodes an implicit contract — "convert" means make editorial decisions, not transcribe — that OpenCode's and Copilot's prompts do not carry.

### 2. Vision capability determines a quality ceiling the harness cannot raise

A good harness pushed DeepSeek-V4-Pro to B+ work from text-only input. But structural errors tied to visual layout ambiguity — wrong heading nesting, paragraphs detached from their headings — persisted across every text-extraction run. These are exactly the errors a visual scan of the source resolves instantly.

### 3. OpenCode's vision routing is model-specific and inconsistently applied

OpenCode correctly sent PDF pages as images to Sonnet 4.6 but returned `"ERROR: Cannot read pdf (this model does not support pdf input)"` for Kimi K2.6 — despite Kimi K2.6 being vision-capable. The capability registry lives in [`sst/models.dev`](https://github.com/sst/models.dev); Kimi's TOML entry lists `input = ["text", "image", "video"]` but omits `"pdf"`, which is the gate `transform.ts` checks. This is a harness configuration gap, not a model limitation.

### 4. Failure recovery separates good agents from bad ones

When the PDF read tool failed, DeepSeek/OpenCode silently accepted degraded output. Kimi/OpenCode installed a better library. DeepSeek/Claude Code installed a better library *and* read the PDF metadata dictionary for provenance fields (`pdf_creator`, `pdf_producer`). The willingness to diagnose and recover rather than pass garbage input downstream is a meaningful agentic capability differentiator.

### 5. Extraction library choice has material impact

Raw binary PDF text extraction is not viable for documents with OpenType ligature fonts — `traffic` becomes `tra  c` or `trac`. Of the libraries used, `pdfplumber` and `PyMuPDF (fitz)` handled both PDFs cleanly. `pymupdf4llm` had a font encoding failure on the InstaTunnel PDF, and Copilot's raw extraction dropped ligature characters entirely.

### 6. Copilot as a null-harness baseline reveals intrinsic model reasoning quality

Copilot provides no behavioral shaping and routes PDFs to no model as images — every model worked from text extraction alone. This surfaces intrinsic reasoning quality most directly:

- **Sonnet** diagnosed the problem and built an 8-iteration custom pipeline: font-size analysis for heading hierarchy, Y-gap measurement for paragraph detection, per-span ligature mapping discovered by context analysis. Result: **B** — correct headings, clean ligatures, still limited by PDF column-wrap truncation.
- **Kimi** produced an instant raw dump with a hallucinated `source_url: "https://blog.unnic.com"` — the only fabrication in the entire benchmark. Result: **F**.
- **DeepSeek** produced an identical instant raw dump with the PDF filename as the document title. Result: **F**.

The within-harness gap between Sonnet/Copilot (B) and Kimi/Copilot (F) is larger than any single harness-driven quality difference. Model intrinsic reasoning is its own axis, independent of vision and harness shaping.

---

## Practical Recommendation

For PDF-to-Markdown conversion tasks:

1. **Prefer a vision-capable model** through a harness that correctly routes PDFs as images — this is the only path that reliably resolves layout-dependent structure.
2. **If text extraction is unavoidable**, use `pdfplumber` or `PyMuPDF (fitz)` over auto-formatting wrappers, and prompt explicitly for editorial conversion rather than transcription.
3. **Verify harness capability detection** before assuming a model's output quality reflects its actual capability ceiling.
4. **Model intrinsic reasoning quality matters** independently of tooling — when no harness scaffolding is available, only Sonnet self-diagnosed and compensated.

---

## Supplementary Runs

Three additional runs were collected under non-standard conditions and are graded for reference only.

| Run | Condition | Grade | Key observation |
|---|---|:---:|---|
| DeepSeek-V4-Pro / OpenCode v2 | fitz pre-installed by prior runs | **B−** | Confirms heading hierarchy error is model-level, not tool-level |
| GPT-5.4 / Codex v1 | Attempted to read existing outputs; re-isolated; different harness | **D+** | Page dump; best provenance frontmatter of any text-extraction run (sha256, conversion timestamp); unique ligature mapping: `fi`→`-`, `fl`→`"` |
| GPT-5.4 / Codex v2 | Explicit follow-up prompt requesting vision-guided reconstruction | **C+** | Correct heading hierarchy via font-size inference; unique figure notes; body text broken into per-line paragraphs with unhealed column-wrap truncations; ~10 min runtime |

---

## Output Files

| Directory | Model | Harness |
|---|---|---|
| `claudecode-sonnet/` | Claude Sonnet 4.6 | Claude Code |
| `opencode-sonnet/` | Claude Sonnet 4.6 | OpenCode |
| `copilot-sonnet/` | Claude Sonnet 4.6 | GitHub Copilot |
| `claudecode-kimi_k2.6/` | Kimi K2.6 | Claude Code |
| `opencode-kimi_k2.6/` | Kimi K2.6 | OpenCode |
| `copilot-kimi/` | Kimi K2.6 | GitHub Copilot |
| `claudecode-deepseek/` | DeepSeek-V4-Pro | Claude Code |
| `opencode-deepseek/` | DeepSeek-V4-Pro | OpenCode |
| `copilot-deepseek/` | DeepSeek-V4-Pro | GitHub Copilot |
| `opencode-deepseek-v2/` | DeepSeek-V4-Pro | OpenCode (tools cached) |
| `codex-gpt_5.4/` | GPT-5.4 | OpenAI Codex |
