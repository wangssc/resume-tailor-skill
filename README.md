# Resume Tailor

> Turn a job description and your real material into a **targeted, one-page A4 resume**.
> An AI agent skill built on the open [Agent Skills](https://agentskills.io) protocol.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue.svg)](https://www.python.org/)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-compatible-brightgreen.svg)](https://agentskills.io)

---

## The problem

Ask an AI to "write me a resume for this JD" and you usually get one of five failures:

- it invents experience, numbers, or certificates
- it pads with generic filler
- the result overflows one page and it silently shrinks the font
- it leaves the template's sample name in place
- you end up editing HTML for an hour because the wording was wrong

Resume Tailor turns this into a **checked pipeline** instead of a single prompt.

| Failure | What this skill does instead |
| --- | --- |
| Invents experience, numbers, credentials | Writes only what is in your material; flags missing hard requirements as risk |
| Fills gaps by guessing | Lists every gap at once (name / contact / education) and asks you to decide |
| Endless HTML editing | **Writes the copy first, gets approval, then fills the template** — editing text costs 1/10 to 1/20 of editing HTML |
| Forgets to replace sample content | Templates are **empty skeletons**: layout and section headings only, every body container blank |
| Content spills to page two | `check_all.py` renders and counts pages; overflow fails loudly |
| PDF has phantom margins or headers | Standard A4 `@page` rules built in; print settings documented |

Core principle: **all iteration happens in the text draft; the HTML is written exactly once.**

---

## Install

This skill follows the open [Agent Skills](https://agentskills.io/specification) protocol and runs in any skills-compatible agent runtime.

### Option A — let the agent install it

```text
Install this skill for me: https://github.com/wangssc/resume-tailor-skill
```

### Option B — clone it manually

| Runtime | Install path |
| --- | --- |
| Claude Code | `~/.claude/skills/resume-tailor/` |
| Codex CLI | `~/.codex/skills/resume-tailor/` |
| Doubao | `~/.user_skills/resume-tailor/` |
| Other runtimes | that runtime's `skills/` directory |

```bash
git clone https://github.com/wangssc/resume-tailor-skill.git ~/.claude/skills/resume-tailor
```

Even without skill auto-loading, you can paste the contents of `SKILL.md` straight into a conversation — it is just Markdown with YAML frontmatter.

### Verify the install

```bash
cd ~/.claude/skills/resume-tailor
python scripts/check_layout.py --self-check
```

Expected:

```text
template-001: [OK] 1 page, 209.9x297.0mm (A4)
template-002: [OK] 1 page, 209.9x297.0mm (A4)
template-003: [OK] 1 page, 209.9x297.0mm (A4)
[OK] all 3 templates are single-page A4.
```

If it returns `[SKIP]` because Chrome/Edge is missing, layout checking and PDF export are unavailable — **everything else still works**.

---

## How it works

### 1. Prepare the input

| Item | Notes |
| --- | --- |
| Job description | Full JD text, or a `.txt` file |
| Material folder | Old resumes, project notes, skill lists, certificate images — `.txt / .md / .docx / .pdf / images` |
| Output directory | Suggested naming: `company-role` |
| Photo | Optional. Without one, each template's default photo is kept (the photo block is never deleted) |
| Target pages | Optional, default 1 |

### 2. Start the conversation

```text
Build a targeted resume for this JD.

<paste the full JD here>

My material folder: D:\job\my-material
Output directory:  D:\job\acme-java-backend
Photo:             D:\job\photo.jpg   (use the template default if absent)
Target length:     1 page
```

### 3. The agent reads the material and reports gaps

It extracts only **verifiable facts** — education, dates, schools, projects, responsibilities, skills, certificates, numbers.

When a hard requirement is missing (name, contact, education), it lists **all** gaps at once and offers three choices:

1. fill them in now
2. run with placeholders, edit the file later
3. explicitly skip that block

It does not invent. When two sources contradict each other, it asks instead of picking one.

### 4. Review the text draft — the step that matters

Before touching any HTML, it produces a **template-independent text draft** for line-by-line review, plus a rough length estimate against the target page count.

- Approve → continue
- Change something → it rewrites **only the draft**, never the HTML

Editing one word after rendering costs 10–20× more than editing it here.

### 5. Pick a template

Three templates are listed in plain text with code, style, and best-fit roles. Reply with the code.

Why choose layout last? The three templates hold different amounts of content — the same draft may fit one page in `003` and spill in `002`. Fixing content first, then layout, minimises rework.

### 6. Fill and validate

```bash
python scripts/check_all.py "/path/to/resume.html" --expect 1
```

`--expect` is the page count you confirmed in step 1. It runs:

1. **Static check** — empty body containers, leftover placeholders, broken image references
2. **Layout check** — rendered page count and page size

On overflow it trims wording; it does not touch font size or margins. On success you get a single deliverable: `resume.html`.

### 7. Export PDF

**Manual (recommended, zero dependencies):**

1. Open `resume.html` in Chrome / Edge
2. `Ctrl+P` (`Cmd+P` on macOS)
3. Destination → **Save as PDF**
4. Paper → **A4**
5. Margins → **None**
6. **Uncheck "Headers and footers"**

Steps 4–6 matter: any non-zero margin pushes content to a second page, and headers/footers print the title and URL.

**Or let the script do it:**

```bash
python scripts/html_to_pdf.py "/path/to/resume.html"
```

It drives the system Chrome/Edge and writes `resume.pdf` next to the HTML.

---

## Templates

| Template | Style | Best for |
| --- | --- | --- |
| `template-001` | Grey band, traditional | State-owned enterprises, public institutions, traditional industries; supports fields like ethnicity, political affiliation, height |
| `template-002` | Blue arrows, modern | Admin, operations, marketing, HR — clean and design-forward |
| `template-003` | Dark blue icons, technical | Engineering roles (Java / backend / internet) — name-forward, minimal |

Previews show the **filled** result. The `resume.html` files in this repo are empty skeletons.

- `assets/template-001/preview.jpg`
- `assets/template-002/preview.jpg`
- `assets/template-003/preview.jpg`

---

## Commands

```bash
# 1) Create a working copy from a template (optionally inject a photo)
python scripts/init_from_template.py --template template-003 \
    --output "/path/to/out" --name resume --photo "/path/to/photo.jpg"

# 2) Static + layout validation in one shot
python scripts/check_all.py "/path/to/out/resume.html" --expect 1

# 3) Export PDF (or just use Ctrl+P)
python scripts/html_to_pdf.py "/path/to/out/resume.html"
```

| Script | Purpose | Common flags |
| --- | --- | --- |
| `check_all.py` | **Recommended entry point**: static + layout | `--expect N` `--browser PATH` `--self-check` |
| `init_from_template.py` | Copy an empty skeleton to the output directory | `--template` `--output` `--name` `--photo` |
| `check_resume.py` | Static check: empty containers, placeholders, broken refs | — |
| `check_layout.py` | Layout check: page count and page size | `--expect` `--pdf` `--browser` `--self-check` |
| `html_to_pdf.py` | Render A4 PDF via the system browser | `--pdf` `--browser` |

**Exit codes**: `0` pass / `1` validation failed / `2` bad input / `3` environment cannot verify (no browser — manual check required).

Code `3` is deliberate: **when the environment cannot verify, the tool never prints `[OK]`**. No "looks like it passed".

---

## FAQ

**One file or two?**
The conversation delivers `resume.html` only. It references `assets/photo.jpeg` by relative path — move or share the `assets/` folder alongside it or the photo breaks.

**Why is my PDF two pages / why does it have a header?**
Set margins to **None** and uncheck **Headers and footers**. The template ships `@page { size: A4; margin: 0 }`, but the browser's print dialog wins.

**Can I get a .docx?**
Not in this version. The docx originals in `sources/` keep their content inside Word text boxes (`txbxContent`), which cannot be rewritten reliably by script. Copy the text out of the HTML into Word instead.

**Can the target be two pages?**
Yes, `--expect 2`. Note that pagination is not balanced — content usually breaks mid-item. One page is the norm for campus and entry-level hiring.

**No Chrome or Edge on this machine?**
Layout checking returns `[SKIP]` (exit 3) and PDF export is unavailable. Open the HTML in any browser and check by eye; static validation and template filling still work. You can also point `--browser` at any Chromium-based browser.

**Can I use my own photo?**
Yes — `--photo <path>` overwrites `assets/photo.jpeg` in the working copy. Without it, the template default stays and the photo block is never removed.

---

## Requirements

- **Python 3.8+** — every script uses the standard library only. No `pip install`.
- **Chrome / Edge / Chromium** — needed only for PDF export and layout checking. Common install paths on Windows / macOS / Linux are auto-detected; `--browser` overrides.
- **pypdf (optional)** — if present, `check_layout.py` prefers it for reading PDFs; otherwise it falls back to a built-in parser.

---

## Repository layout

```text
resume-tailor/
├── SKILL.md                       # Skill entry: triggers and the end-to-end workflow
├── README.md
├── LICENSE
├── assets/                        # Empty-skeleton templates, all single-page A4
│   ├── template-001/
│   │   ├── resume.html            # Skeleton: layout + headings, all body containers empty
│   │   ├── preview.jpg            # Filled-state preview
│   │   └── assets/photo.jpeg      # Default photo
│   ├── template-002/
│   └── template-003/
├── sources/                       # Original docx templates (redesign only, not used in generation)
├── references/                    # Methodology docs, loaded on demand
│   ├── example-template-00X.html  # Filled examples: for density reference, do not copy
│   ├── jd-analysis.md             # JD decomposition and keyword matching
│   ├── resume-writing.md          # Truthfulness rules, STAR, quantification
│   └── template-guide.md          # Template structure and layout-protection rules
├── scripts/                       # Deterministic scripts, stdlib only
│   ├── init_from_template.py
│   ├── check_all.py
│   ├── check_resume.py
│   ├── check_layout.py
│   └── html_to_pdf.py
├── examples/                      # Demo screenshots and a fictional sample material set
└── image/                         # Screenshots used by this README
```

---

## Known limits

- **No docx export** — see the FAQ for why.
- **Layout checking needs a local browser** — without Chrome/Edge you get a manual check and an explicit `[SKIP]`, never a fake pass.
- **Three templates only**, all based on Chinese resume conventions. English resumes and academic CVs are out of scope.
- **No application submission** — it produces files; it does not fill in job-portal forms.

---

## License and credits

Released under the **MIT License** (see [LICENSE](LICENSE)). Use, copy, modify, and distribute freely for personal or commercial purposes, keeping the copyright and license notice.

The resume templates are **redesigned from** [mmmlllnnn/ResumeCollection](https://github.com/mmmlllnnn/ResumeCollection) (MIT). Template design credit belongs to the original author; this project re-creates them as HTML and builds the JD-driven generation workflow on top. Thanks to the original collector for sharing.

---

## Maintainer notes

<details>
<summary>Regenerating preview images after a template change (not needed by users)</summary>

1. After editing `assets/template-00X/resume.html`, confirm it is still single-page A4:

   ```bash
   python scripts/check_layout.py --self-check
   ```

2. Convert the HTML to PDF, export page 1 as an image, and overwrite `preview.jpg`.

3. If `examples/` screenshots need updating, export them from a real render too — **do not hand-screenshot**, since window size and zoom make them inconsistent with script output.

</details>
