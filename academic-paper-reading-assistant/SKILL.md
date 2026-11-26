---
name: "academic-paper-reading-assistant"
description: "Turn a technical paper (pure maths, theory, ML or other empirical work; arXiv link/id, PDF and/or LaTeX source) into one reading-copy PDF: a front reading guide (key references, argument graph, reading route, exercises) plus the paper with route sections boxed."
---

# Paper reading copy

The user supplies a paper as a PDF, an arXiv source tarball, an arXiv link or id, a title, or any mix. If they don't attach the LaTeX source, fetch it yourself (Stage 0) rather than asking them to upload it. They want a **single PDF** they can put on an iPad or print. It has two parts:

1. **Reading guide**, a few pages at the front: the idea (a plain paragraph, then a precise statement), how to use the copy, the proof (or, for an empirical paper, the argument) in a few steps, the key references to read first (at most 5, often none), an argument graph that includes those references, a numbered reading route with page ranges, an optional variants or approaches table, a claims-and-evidence table for empirical papers, a "safe to skip" list, and self-check questions with answers on the last page.
2. **The paper, recompiled from source with its layout unchanged.** Every section on the reading route is boxed and tagged with its route number. Yellow means core, and blue means know the statement (in a theory paper) or the gist (in an empirical one).

The skill works for any technical paper. Stage 1 starts by deciding the paper's **type**, which fixes what "result", "steps", "core" and the exercises mean. Everything from Stage 2 onwards is the same for every type.

The PDF viewer's page labels read `Guide i…` for the guide, then the paper's own page numbers. All the paper's internal links still work.

Create a task list, since this is multi-stage work. Keep untrusted input separate: unpack the tarball into its own empty directory, keep your scripts in another directory, and run Python on its files with `-I`.

## Conventions (use the same ones everywhere)

| Level | Meaning | Colour |
|---|---|---|
| core | read closely and redo the computations | fill RGB 255,221,51 (yellow), line RGB 214,168,0 |
| know | know the statement, skim the proof (empirical: know the gist and key formula, skim the detail) | RGB 47,95,168 (blue) |
| skip | safe to skip on a first pass | dashed grey, not boxed in the paper |
| result | the main theorem, or the headline empirical result (graph only) | dark ink fill, white text |
| ref: essential | a cited paper you must read first (guide and graph only) | grey fill, thick ink border, square corners |
| ref: context | a cited paper that helps but is not required (guide and graph only) | white fill, dotted grey border, square corners |

Route numbers follow the **logic of the argument**, not the order of the paper: the logic of the proof for a theory paper, and for an empirical paper the design argument (problem → mechanism → objective or training signal → evidence). The numbers in the guide match the tags on the boxes.

## Stage 0: get the source and the PDF yourself

Do this whenever the user gives a link, an arXiv id, a title, or only a PDF, and has not attached the LaTeX source. Skip it for anything they attached.

1. **Find the arXiv id.** Take it from a URL (`arxiv.org/abs|pdf|html/<id>`), from `arXiv:<id>`, or from a DOI of the form `10.48550/arXiv.<id>`. For a title or a non-arXiv link, WebSearch for `<title> arXiv` and confirm the title and authors on the abs page with WebFetch. Keep the version suffix (`v2`) if the user gave a PDF or link with one, so the source matches their PDF. Otherwise use the latest version.
2. **Download into new, empty directories**, one per download, with curl through the session proxy. Make one request per URL, with `-L --fail -A "paper-reading-assistant (personal use)"`. Don't loop, retry in bursts, or parallelize. One paper is two requests:
   - source: `https://arxiv.org/e-print/<id>`
   - PDF, only if the user gave none (Stage 2 needs it for the page-count check): `https://arxiv.org/pdf/<id>`
3. **Identify what came back** with `file`. It is one of: a gzipped tar (the usual case), a single gzipped `.tex`, or a PDF. A PDF means the authors submitted no source, so use the fallback at the end of this skill. For a tar, list it first with `tar tf` and refuse the archive if any path is absolute or contains `..`. Extract with `tar xf <file> -C <dir> --no-same-owner`. For a single gzipped file use `gunzip -c`. Then continue at Stage 2 (`00README.json`, compile, page-count check).
4. **If arXiv has no source, or the paper isn't on arXiv**, look in this order: a GitHub repo or project page linked from the abs page or the paper (clone it with `gh repo clone` or `git clone`, and look for the `.tex` files and the class and style files they need); then the authors' own page if it links a source archive. Publisher sites, OpenReview, the ACL Anthology, bioRxiv and medRxiv give PDFs, not LaTeX, so for those use the PDF fallback.
5. **If a fetch is refused** (403 or 407 from the proxy, "CONNECT tunnel failed", host not allowed), that is the workspace's network policy. Don't retry, and don't go round it with mirrors, caches or other tools. Say so in one line and ask the user to attach the tarball (on the abs page: Access Paper, then TeX Source). Meanwhile, continue Stage 1 from whatever PDF or text you do have. WebFetch on `arxiv.org/abs/<id>` or `arxiv.org/html/<id>` may give enough text to draft the plan.
6. **Say what you used** in the final message: the arXiv id and version, or the user's upload, and whether the page count matched. This is a personal reading copy, so remind the user not to redistribute it if the paper's license is restrictive.

## Stage 1: understand the paper and write the reading plan

0. **Decide the paper's type.** Most papers are one of these, or a mix. The rest of this stage uses the column that fits. A mixed paper (for example, a theory section followed by experiments) uses the theory column for its theorems and the empirical column for its experiments.

   | | Theory (pure or applied maths, TCS, theoretical physics or stats) | Empirical / method (ML, systems, applied science) |
   |---|---|---|
   | units to inventory | Theorem, Proposition, Lemma, Definition, Assumption | the method's components, numbered equations, Figures and Tables, `\paragraph` run-in headings, experiments |
   | `result` node | the main theorem | the headline claim (for example, first on a benchmark), with the figure or table that shows it |
   | "steps" section | The proof in N steps | The argument in N steps (problem → mechanism → objective → evidence) |
   | "Precisely" block of The idea | the main theorem's statement and the one idea that makes the proof work | the key mechanism (a mask, an objective, an update rule) and its immediate consequences |
   | core | where the new idea lives and where the main proof turns | the new mechanism, and the prior approach it fixes when that is the paper's motivation |
   | know | supporting estimates, statements you use, one model application | architecture and training detail, the objective, the headline results |
   | skip | technical appendices whose statement is enough, extra applications | dataset recipes, hyper-parameter lists, full result tables, latency and engineering sections (unless the reader deploys), contributor lists |
   | parallel-cases table | bulk/edge, real/complex, discrete/continuous | the approaches compared (prior designs vs this one) on the properties the paper argues about |
   | exercises | special cases, small computations | redo a figure's bookkeeping (a mask, a token count, a shape), a small numeric check of a loss or metric, and design the missing ablation |
   | extra section | — | **Claims and evidence** (step 9) |

   Surveys and position papers have no single result. Make the route follow the paper's taxonomy, and use the claims-and-evidence table for its strongest claims.

1. Run `pdftotext -layout` on the paper. For theory, list every Theorem, Proposition, Lemma, Definition and Assumption with its line number (grep `^(Theorem|Proposition|Lemma|...) [0-9A-F]+\.[0-9]+`). For empirical work, list the `Figure N:` and `Table N:` captions and the numbered equations. In both cases list the section headings with their pages. Once you have the source, grep it for `\section`, `\subsection` and `\paragraph`: run-in paragraph headings are often where an empirical paper's units start and end.
2. Read in full: the abstract, the introduction, the main-results subsection, and the proof-ideas or outline subsection (or the method section, for empirical work). Then read the sections the outline calls key, especially the proof of the main theorem (or the method's definition, with its key figure) and anything named as the new idea.
3. Build the dependency graph. For theory, show the hypothesis or setting, the steps of the main proof, the main theorem (mark it as `result`), any corollaries or criteria, and the applications. For empirical work, show the framing, the prior approach and its flaw, the new mechanism and its parts, the objective, and the headline result. Draw an edge for "is used by".
4. Classify each node using the table in step 0:
   - **core** is where the new idea lives, and where the main argument turns. Usually 3–4 items.
   - **know** covers what you need to use but not rebuild.
   - **skip** covers material whose summary is enough.
5. For each route item, write what to take away: the key formula, the trick in one or two sentences, and an "aha" line where one is earned. Give exact page ranges, and say exactly where each box starts and ends (for example, "§4.1 up to the end of Corollary 4.2").
6. **Triage the bibliography for key references.** The guide has a "Key references" section listing at most 5 cited works, each tagged **essential prerequisite** or **useful context**. Its value depends entirely on its signal-to-noise ratio: a reader who sees "essential prerequisite" will go and read that paper before this one, so the tag must only appear when that is truly worth their time. An empty section is a perfectly good, and common, outcome.

   Judge each citation mainly by **how this paper uses it**, which is usually obvious from the text around the `\cite`. Grep the source for every `\cite` and look at the ones inside the outline, the proofs of route items, and phrases such as "we follow", "as in", "the proof is identical to", "inspired by", "see [X] for details", "using [X, Theorem 2.1]".
   - **Essential prerequisite**: a step on the reading route rests on the reference, and the paper neither restates nor reproves what it takes from it, so the reader cannot follow the *argument* of that step without opening the reference. Typical cases: the core construction is "as in [X, Section 3]" and never explained here, or the paper assumes the reader already works in a framework set up in [X]. Having to take a deferred technical estimate on trust ("the proof is the same as [X, Lemma 5.3], which we omit") is not enough on its own; that is useful context. Expect 0–2 essential references per paper.
   - **Useful context**: the paper's new idea is adapted from the reference, or a core step defers routine details to it, but the paper can be followed on its own. Reading it deepens understanding of the route.
   - **Leave out**: related-work and history citations, results cited for comparison, standard textbook facts used as black boxes (a large-deviation principle, a standard formula), statements the paper quotes in full (for example in an appendix), and applications or extensions.
   - **When in doubt, demote.** Between essential and context, choose context; between context and leaving it out, leave it out. Never pad the list to reach 5.

   For each kept reference, record: the short citation (authors, short title, year, and the arXiv id or journal if the bibliography gives it); one or two specific results or sections **by name or number only**, taken from how the paper cites them (`[X, Section 5]`, `[X, Lemma 5.3]`, "the recursive moment estimate"); and one line on which route items use it and why. Don't invent result numbers: if the paper cites the work only as a whole and you can't confirm a section, name the topic instead. Where WebSearch/WebFetch work, you may look up the abstract or table of contents to confirm the section names and your judgement, but this is optional and the paper's own usage should decide the tag. If a lookup is refused, carry on without it.

   Add each kept reference to the dependency graph as a **leaf node** (no incoming edges), with an edge into every main-paper node that uses it.

7. Write 4–6 self-check questions. At least one should be a small concrete computation: a special case of the main formula, or for empirical work, redoing a figure's bookkeeping (which tokens a mask lets through, a token or parameter count, a loss value for given numbers). For empirical papers, also include one critical question, such as "what experiment would isolate X?". **Check every answer you give independently**, by a separate calculation or a numeric check. If you can't check one, word it as "should give…" or drop it. If a question needs a quantity the paper doesn't give (a sampling distribution, a hyper-parameter), say it is hypothetical in the question.
8. If the paper has parallel cases (bulk/edge, real/complex, discrete/continuous), or compares its approach against several prior designs, make a side-by-side table of what changes.
9. **Empirical papers: claims and evidence.** Make a table with three columns: claim, evidence (the figure, table or section), and strength or caveat. Include each contribution the abstract or introduction lists. Mark plainly where a claim is argued but not measured: no ablation isolating the new component, baselines re-run by the authors, training data not named, possible overlap between training and test data not discussed, or a comparison on one axis only (for example, parameters but not latency). Grep the text before saying something is absent (`ablat`, `leak`, `contamin`, `overlap`), and phrase it as "the report does not…", not as an accusation.
10. **Write "The idea" last, once the route is settled.** It is the guide's opening section and the part most readers judge the guide by. It has two parts.
    - **A plain paragraph of about 5–7 short sentences: problem → fix → cost → payoff.** Write one idea per sentence, conversational but exact. Define every acronym and every term the reader needs on first use.
    - **A "Precisely." block** in the paper's own notation. It states the key object exactly (see the step-0 table) and then gives its 2–4 immediate consequences as a numbered list. This is what the reader checks against the key figure or theorem while reading the route.

    Rules:
    - **Strongly prefer a self-contained explanation.** Name prior methods by what they do ("hiding input patches so the model must predict across the gap"), not by citation. A reference is allowed when it is genuinely the clearest explanation and the work is very well known (a standard Transformer decoder, the Kalman filter, Wigner's semicircle law). Never require the reader to have read a specific recent paper to follow the section.
    - **No performance numbers.** Leave out benchmark scores, ranks, percentage gains, win rates and model-size comparisons. They clutter the section and are orthogonal to understanding the paper; they belong in the results route item and the claims-and-evidence table.
    - **Leave out secondary components** (normalisation, auxiliary heads, covariate handling, training data) and any positioning against related work. Those have their own places in the guide.
    - **Analogies: at most one sentence**, and only when the mechanism is genuinely counter-intuitive. Don't build the paragraph around one.

    Example (Timestral, an ML forecasting paper):

    ```
    A causal Transformer is a natural forecaster: one pass over a series gives a forecast from every time point,
    but it only learns to predict one patch ahead. The usual way to reach further is to hide some input patches, so
    the model must predict across the gap ("contiguous patch masking"). That quietly throws data away: a hidden
    patch is also hidden from every later forecast, so how far ahead you train and how much context you keep are
    tied together. Timestral stops making one token do two jobs. Each patch gets a context token holding its observed
    values and a prediction token that does the forecasting, and context tokens never look at prediction tokens.

    Precisely. Let v_i be the context token of patch i, and p_i a prediction token (one shared learned embedding,
    placed at position i). Training splits the series into forecast blocks of random length. In a block [b, b+k-1]:
    v_i attends to v_0..v_i; p_j attends to v_0..v_{b-1} and p_b..p_j. So p_j forecasts patch j at distance j-b+1
    from all observations before the block. Consequences: (1) the block-length distribution is the horizon
    distribution, and changing it never removes context; (2) "missing" (a gap inside some v_i) and "to forecast"
    (a p_j) are distinct; (3) training costs about 2x tokens, and inference does one block of up to 32 patches in
    one pass.
    ```

If the user just wants the plan quickly, you can share the outline in chat before building the PDF. The PDF is the deliverable.

## Stage 2: compile the original source unchanged

```bash
mkdir -p $W/src && tar xf paper.tar -C $W/src   # or unzip, or gunzip a single .tex.gz
cat $W/src/00README.json  # arXiv says which file is the top level and which compiler to use
```

Copy the source into a build directory and compile twice with the stated compiler. **Check that the page count equals the arXiv PDF's.** If it doesn't, the page references in the guide won't match.

If a package is missing and installs are blocked, see what the paper actually uses from it. Then write a minimal local stub `.sty` that accepts any options (`\DeclareOption*{}\ProcessOptions\relax`) and provides only the commands the paper uses. Tell the user about any visible difference (for example, `dsfont`'s `\mathds{1}` becoming a bold 1).

If there is no source, or it won't compile, use the fallback at the end of this skill.

## Stage 3: box the route sections without changing the layout

Don't wrap sections in `tcolorbox`/`mdframed`, because that moves page breaks. Instead, record invisible positions with `zref` and draw the boxes behind the text, in the shipout background.

### 3a. Preamble (insert just before `\begin{document}`)

```latex
\usepackage[savepos]{zref}
\usepackage{eso-pic}
\makeatletter
\zref@newprop*{pgnum}[0]{\the\c@page}   % starred = expanded at shipout, so the page is correct
\zref@addprop{savepos}{pgnum}
\newcommand\hlmark[1]{\zsavepos{hl:#1}}
\AtBeginDocument{\typeout{HLGEOM \the\paperwidth\space\the\paperheight\space\the\topmargin\space\the\headheight\space\the\headsep\space\the\textheight\space\the\oddsidemargin\space\the\textwidth}}
\definecolor{hlYfill}{RGB}{255,221,51}\definecolor{hlYline}{RGB}{214,168,0}
\definecolor{hlBfill}{RGB}{47,95,168}\definecolor{hlBline}{RGB}{47,95,168}
\InputIfFileExists{hlboxes.tex}{}{}
\AddToShipoutPictureBG{\ifcsname hlpage\the\c@page\endcsname\csname hlpage\the\c@page\endcsname\fi}
\makeatother
```

This needs `xcolor` and `tikz`. Add `\usepackage{tikz}` if the paper doesn't load it.

### 3b. Marks

Insert them with a small Python script that edits by line number and **asserts the expected text is on each line**. Don't use blind `sed`.

- **Start of a region:** append `\hlmark{hX}` to the end of the `\subsection{…}` line, after the heading (and after any `\label`). Don't put marks inside heading arguments, because they would be copied into the TOC. For a run-in `\paragraph{…}` heading, which is a common unit in ML papers, append the mark right after the closing brace. It is recorded just above the paragraph's first line, so it needs a negative `top_off` (see 3c).
- **Region ending at the next heading:** use that heading's own mark as the end.
- **Region ending mid-section** (after a theorem statement or a proof): insert `\hlmark{rXe}` inline, straight after the final sentence's last token. It must be in horizontal mode, so not after a display or right after `\end{…}`.

Adjacent regions can share a heading mark. For example, §6.2 ends where §6.3 starts.

### 3c. Compile twice, then calibrate the offsets

Where a mark sits relative to its heading depends on the document class. Measure it:

```bash
grep 'hl:' paper.aux                     # posy in sp (divide by 65536 for pt, measured from the page bottom)
pdftotext -f P -l P -bbox paper.pdf - | grep '>6\.3<'   # heading glyph box, in bp from the top
```

Convert both to the same frame (bp ≈ pt; y_from_bottom = page height − y_from_top). In `article`, the mark sat about 11pt below the heading baseline. The heading glyph top was about 19pt above the mark, and the previous text line's bottom about 39pt above it. Choose:

- `TOP_OFF`: the box top, a few pt above the heading glyph top (25 in the test case).
- `HEAD_END_OFF`: the bottom of a box that ends at the next heading. It must be above that heading's `TOP_OFF` with a gap of at least 6pt, and below the previous text line (33 in the test case).
- `INLINE_END_OFF`: how far below an inline end mark's baseline the box ends (7).

Script this: for each mark, find the nearest heading glyph with `pdftotext -bbox` and print the glyph top and bottom and the previous text line's bottom, all relative to the mark. Then set `top_off` and `end_off` **per box** in `regions.json`, rather than relying on global values. In one NeurIPS-style paper, heading glyph tops sat 10.6–17.2pt above their marks across the same kind of heading, and a run-in `\paragraph` mark sat 7.4pt *above* its glyph top (`top_off` −2.5). The tag is centred on the box's top border, so its lower half reaches about 4.5pt down: with a long heading, raise `top_off` until the tag clears the glyph top, and lower the previous box's `end_off` to keep the 6pt gap. Measure each kind of heading that ends a box separately: a `\section` and a `\subsection` can sit very differently relative to their marks. Run-in subsection headings (an after-skip of 0 or less in `\@startsection`) put the mark on the heading line itself, about at the glyph top, so the offsets shrink (in one test: `TOP_OFF` 4, `HEAD_END_OFF` 11.5, `INLINE_END_OFF` 4), while a display `\section` heading in the same paper sat 19pt above its mark and needed `end_off` 26 for the box ending there.

### 3d. Generate `hlboxes.tex` and compile once more

Save this as `make_boxes.py` in your scripts directory. Run `python3 -I make_boxes.py paper.aux paper.log regions.json hlboxes.tex 25 33 7`.

`regions.json` lists one object per box: `{"start":"h51","end":"h52","end_kind":"heading","color":"Y","label":"2 $\\cdot$ Linearization: the key new idea"}` (`end_kind` is `heading` or `inline`; `color` is `Y` or `B`; optional `end_off` and `top_off` override the offsets for that one box). Put one object for page 1 under the key `"legend"` as a string. It is a one-line note telling readers what the colours mean and listing the route sections. An optional top-level `"kinds"`, for example `{"Y": "core, read closely", "B": "know the gist, skim"}`, sets the tag wording; use it for empirical papers, where "skim proof" would be wrong. `"legend_pos"` is `"margin"` (the default: the legend sits in the top margin, above the title) or `"text"` (the old placement inside the text block, which covers the title on most first pages).

```python
import re, sys, json, collections
aux, log, spec, out = sys.argv[1:5]
TOP_OFF, HEAD_END_OFF, INLINE_END_OFF = map(float, sys.argv[5:8])
pos = {m[1]: (int(m[4]), int(m[3]) / 65536) for m in re.finditer(
    r'\\zref@newlabel\{hl:(\w+)\}\{\\posx\{(\d+)\}\\posy\{(\d+)\}\\pgnum\{(\d+)\}\}', open(aux).read())}
g = re.search(r'HLGEOM (.*)', open(log, errors='ignore').read())[1].split()
pw, ph, tm, hh, hs, th, osm, tw = [float(x.replace('pt', '')) for x in g]
ttop = ph - (72.27 + tm + hh + hs); tbot = ttop - th
x0, x1 = 72.27 + osm - 9, 72.27 + osm + tw + 9      # single-column text block +-9pt
CT, CB = ttop + 4, tbot - 6                          # continuation-page top/bottom
S = json.load(open(spec)); pages = collections.defaultdict(list)
for r in S['regions']:
    sp, sy = pos[r['start']]; ep, ey = pos[r['end']]
    top = min(sy + r.get('top_off', TOP_OFF), CT)
    if r['end_kind'] == 'heading':
        bot = ey + r.get('end_off', HEAD_END_OFF)
        if bot > ttop - 10: ep, bot = ep - 1, CB      # next heading opens a page: stop on the previous one
    else:
        bot = ey - r.get('end_off', INLINE_END_OFF)
    for p in range(sp, ep + 1):
        pages[p].append((r['color'], top if p == sp else CT, bot if p == ep else CB, p == sp, p == ep, r['label']))
def box(c, t, b, first, last, lab):
    fill, line, txt, op = ('hlYfill', 'hlYline', 'black', '0.20') if c == 'Y' else ('hlBfill', 'hlBline', 'white', '0.07')
    L = lambda a, b_: f'\\draw[{line},line width=1.2pt] {a} -- {b_};'
    o = [f'\\fill[{fill},fill opacity={op}] ({x0:.2f},{b:.2f}) rectangle ({x1:.2f},{t:.2f});',
         L(f'({x0:.2f},{b:.2f})', f'({x0:.2f},{t:.2f})'), L(f'({x1:.2f},{b:.2f})', f'({x1:.2f},{t:.2f})')]
    if first: o.append(L(f'({x0:.2f},{t:.2f})', f'({x1:.2f},{t:.2f})'))   # open edges at page breaks
    if last:  o.append(L(f'({x0:.2f},{b:.2f})', f'({x1:.2f},{b:.2f})'))
    kind = S.get('kinds', {}).get(c, 'core, read closely' if c == 'Y' else 'statement, skim proof')
    tag = f'{lab} \\quad\\textmd{{({kind})}}' if first else f'{lab.split(" ")[0]} (continued)'
    o.append(f'\\node[anchor=east,fill={fill},text={txt},font=\\sffamily\\scriptsize\\bfseries,inner sep=2.2pt,rounded corners=1.5pt] at ({x1-8:.2f},{t:.2f}) {{{tag}}};')
    return o
anchor, ly = ('south west', ttop + 14) if S.get('legend_pos', 'margin') == 'margin' else ('north west', ttop + 8)
legend = (f'\\node[anchor={anchor},draw=black!50,fill=white,font=\\sffamily\\scriptsize,inner sep=4pt,'
          f'text width={x1-x0-8:.1f}pt,align=left] at ({x0:.2f},{ly:.2f}) {{{S["legend"]}}};')
pages[1] = pages.get(1, [])
with open(out, 'w') as f:
    for p in sorted(pages):
        body = [l for r in pages[p] for l in box(*r)] + ([legend] if p == 1 else [])
        f.write(f'\\expandafter\\def\\csname hlpage{p}\\endcsname{{\\put(0,0){{\\begin{{tikzpicture}}[x=1pt,y=1pt]\\useasboundingbox (0,0);\n'
                + '\n'.join(body) + '\n\\end{tikzpicture}}}\n')
```

An example legend string: `\\textbf{Annotated reading copy.} \\colorbox{hlYfill}{\\strut\\ yellow\\ } core, read closely \\quad \\colorbox{hlBfill}{\\strut\\color{white}\\ blue\\ } know the statement, skim the proof. Numbers follow the reading route: 1 (\\S1.1--1.2), ...`

### 3e. Check

- The page count is unchanged.
- Every `hl:` label is on the page you expected from the plan.
- No compile errors.

Render every start page and end page with `pdftoppm -r 60` and combine them into one contact sheet with PIL. Then look closely, at about 110 dpi, at one start heading and at one pair of adjacent boxes. Check that:

- no border or tag crosses a heading or a text line;
- adjacent boxes don't touch;
- the inline end sits just under the final line.

If something is wrong, fix the offsets and recompile once. That only regenerates `hlboxes.tex`, so the layout can't move.

## Stage 4: the reading-guide PDF

Write `guide.tex` as a standalone document in its own directory. Use the **paper's own page size**, taken from `HLGEOM`, so the merged PDF is uniform. Number the guide's pages in roman numerals. Tested skeleton:

```latex
\documentclass[10pt]{article}
\usepackage[paperwidth=<pw>pt,paperheight=<ph>pt,margin=0.85in,top=0.75in,bottom=0.85in]{geometry}
\usepackage[T1]{fontenc}\usepackage{mathpazo}\usepackage[scaled=0.92]{helvet}\renewcommand{\ttdefault}{lmtt}  % avoids bitmap T1 typewriter fonts
\usepackage{amsmath,amssymb}\usepackage{xcolor,tikz,enumitem,tabularx,booktabs,array}
\usetikzlibrary{arrows.meta,positioning,calc}
\usepackage{needspace}\usepackage[hidelinks]{hyperref}
\definecolor{hlYfill}{RGB}{255,221,51}\definecolor{hlYline}{RGB}{214,168,0}
\definecolor{hlBfill}{RGB}{47,95,168}\definecolor{hlBline}{RGB}{47,95,168}
\definecolor{ink}{RGB}{24,32,46}\definecolor{muted}{RGB}{93,102,118}\definecolor{edge}{RGB}{122,131,148}
\colorlet{hlYsoft}{hlYfill!28}\colorlet{hlBsoft}{hlBfill!7}
\newcommand{\core}{\colorbox{hlYfill}{\sffamily\scriptsize\bfseries\strut CORE}}
\newcommand{\refess}{\colorbox{ink}{\sffamily\scriptsize\bfseries\color{white}\strut ESSENTIAL PREREQUISITE}}
\newcommand{\refctx}{\fcolorbox{muted}{white}{\sffamily\scriptsize\bfseries\color{muted}\strut USEFUL CONTEXT}}
\newcommand{\keyref}[4]{% #1 \refess or \refctx, #2 short citation, #3 results/sections to look at, #4 why, and which route items use it
  \par\smallskip\needspace{4\baselineskip}\noindent #1\hspace{6pt}{\bfseries #2}\par\nopagebreak
  \begin{list}{}{\leftmargin=12pt\topsep=1pt}\item[]\textit{Look at:} #3.\quad #4\end{list}}
\newcommand{\know}{\colorbox{hlBfill}{\sffamily\scriptsize\bfseries\color{white}\strut STATEMENT}}  % use GIST for empirical papers
\newcommand{\pp}[1]{{\sffamily\footnotesize\color{muted}#1}}
\newcommand{\eyebrow}[1]{{\sffamily\footnotesize\color{muted}\MakeUppercase{#1}}}
\newcommand{\gsec}[1]{\par\bigskip\needspace{10\baselineskip}{\sffamily\Large\bfseries #1}\par\smallskip}
\newcommand{\aha}[1]{\par\smallskip\noindent\hspace*{2pt}\textcolor{hlYline}{\rule[-2pt]{2.5pt}{11pt}}\hspace{6pt}\textit{#1}\par}
\newenvironment{routeitem}[4]{% #1 number, #2 c=core or k=know, #3 title, #4 sections and pages
  \par\medskip\needspace{9\baselineskip}\noindent
  \if c#2\def\rfill{hlYfill}\else\def\rfill{white}\fi
  \begin{tikzpicture}[baseline=(n.base)]\node[circle,draw=ink,line width=0.8pt,fill=\rfill,minimum size=17pt,inner sep=0pt,font=\sffamily\bfseries] (n) {#1};\end{tikzpicture}%
  \hspace{6pt}{\sffamily\bfseries #3}\hspace{8pt}\if c#2\core\else\know\fi\hspace{6pt}\pp{#4}\par\nopagebreak
  \begin{list}{}{\leftmargin=25pt\rightmargin=0pt\topsep=3pt}\item[]}{\end{list}}
\setlist{itemsep=2pt,topsep=3pt}\emergencystretch=2em\setlength{\parindent}{0pt}\setlength{\parskip}{4pt}
\renewcommand{\thepage}{\roman{page}}
% add any of the paper's math macros you quote in the guide (\E, \Var, \im, \ii, ...); use \newcommand, not \renewcommand
\begin{document}
% p.i: eyebrow ("Reading guide · arXiv id"), {\raggedright\fontsize{22}{26}\selectfont\bfseries Title\par}, authors,
%      \gsec{The idea} (plain paragraph, then {\itshape Precisely.} block with a short enumerate of consequences; Stage 1 step 10),
%      \gsec{How to use this copy} (legend swatches drawn with tikz; for an empirical paper add one sentence saying the
%      route follows the design argument and that a claims-and-evidence table is included),
%      \gsec{The proof in N steps} or, for empirical work, \gsec{The argument in N steps}
% p.ii (\newpage): Key references at the top (the plain {\sffamily\Large\bfseries ...} heading, since \gsec's \bigskip
%      wastes space at a page top), with \keyref entries, essential ones first. Always include the heading; if none were kept,
%      write one sentence such as "None: the paper is self-contained for a first read; its citations are background."
%      Then \gsec{The argument at a glance} + graph + small legend. Putting the references on the graph page keeps them
%      next to their leaf nodes and stops a short section spilling onto an almost empty page.
% p.iii+: \gsec{Reading route} with routeitem environments, then the variants or approaches table (tabularx + booktabs),
%      for empirical papers \gsec{Claims and evidence} (tabularx: claim | evidence | strength/caveat, in \small),
%      Safe to skip, Check your understanding
% last page (\newpage): \gsec{Answers}
\end{document}
```

Graph styles that worked (put them in the `tikzpicture` options). Place the nodes on an explicit grid, for example three columns at x = 0, 5.6, 11.2 cm and rows about 2.3–2.6 cm apart:

```latex
font=\sffamily\small, >={Stealth[length=6pt]},
box/.style={rounded corners=3pt,align=center,inner sep=5pt,text width=#1,line width=1pt}, box/.default=4.3cm,
core/.style={box=#1,fill=hlYfill,draw=ink}, core/.default=4.3cm,
know/.style={box=#1,fill=white,draw=hlBline}, know/.default=4.3cm,
skip/.style={box=#1,fill=white,draw=edge,dashed,text=muted}, skip/.default=4.3cm,
res/.style={box=#1,fill=ink,draw=ink,text=white}, res/.default=6.8cm,
refess/.style={box=#1,rounded corners=0pt,fill=muted!14,draw=ink,line width=1.6pt}, refess/.default=3.6cm,
refctx/.style={box=#1,rounded corners=0pt,fill=white,draw=muted,dotted,line width=1.1pt}, refctx/.default=3.6cm,
re/.style={->,draw=muted,dotted,line width=0.9pt},
e/.style={->,draw=edge,line width=0.9pt}, lbl/.style={font=\sffamily\scriptsize\itshape,text=muted}
```

Node text is three lines: `\textbf{Title}\\ one-line gist\\ {\ttfamily\scriptsize Thm x.y · pp.~a--b}`.

Key-reference nodes are leaves. Put them in their own column on the left or a row along the bottom, away from the main flow, and join them to the nodes that use them with `re` edges, so the eye can tell outside inputs from the paper's own steps. Their text is `{\sffamily\scriptsize ESSENTIAL}` or `{\sffamily\scriptsize CONTEXT}`, then `\textbf{Author--Author 'YY}`, then the result or section looked at. Add both reference styles to the graph's small legend when the graph has any.

Compile twice, render every page at about 55 dpi into one contact sheet, and render the graph page at about 90 dpi. Check for:

- arrows running through boxes (route a `|-` edge into the target's top, not its side, when the source sits above the target);
- hyphenated or crowded node text (widen the node);
- route or section headings stranded at the bottom of a page;
- overfull lines in the log.

## Stage 5: merge and check

Save as `merge.py` and run `python3 -I merge.py guide.pdf paper.pdf out.pdf`. pypdf prints many harmless `Annotation sizes differ` warnings, which you can filter out.

```python
import sys
from pypdf import PdfReader, PdfWriter
guide, paper, out = sys.argv[1:4]
g, p = PdfReader(guide), PdfReader(paper)
w = PdfWriter()
w.append(g, import_outline=False)
w.append(p)                                  # keeps the paper's links, named dests and bookmarks
ng, n = len(g.pages), len(g.pages) + len(p.pages)
w.set_page_label(0, ng - 1, style='/r', prefix='Guide ')
w.set_page_label(ng, n - 1, style='/D', start=1)   # viewer page numbers = the paper's page numbers
w.add_outline_item('Reading guide', 0)
w.add_outline_item('Paper (annotated)', ng)
w.write(out)
```

If the paper's first page isn't numbered 1, set `start` to match. Then check:

- the page count is guide + paper;
- `r.page_labels` is as expected;
- the number of `/Link` annotations on the paper pages equals the count in the unmerged paper;
- no link points to a missing named destination;
- `qpdf --check` is clean;
- a contact sheet of the first `ng + 2` pages looks right.

## Deliver

Name the file `<FirstAuthor>-<SecondAuthor>-<ShortTitle>-reading-copy.pdf`. Send it with SendUserFile (`display: attach`). In two or three sentences, say what's in it and give the route at a glance, plus the key references (or that there were none). Name any stubbed packages and the visible difference each causes. Don't send intermediate files.

## Known pitfalls (all seen in practice)

- **Fill colour from `\ifx` inside TikZ options.** `fill=\ifx c#2 hlYfill\fi` leaves a leading space, which gives "Undefined color ` hlYfill`". Set `\def\rfill{…}` before the picture instead.
- **Brace counting in Python f-strings that write TeX.** The `\csname…\endcsname{\put(0,0){…}}` wrapper must close with exactly `}}` after `\end{tikzpicture}`. A mistake gives "Too many }'s".
- **Page keys.** Boxes are keyed on `\c@page`. If the paper restarts page numbering (for example, roman-numbered front matter), keys collide. In that case key on `\thepage` in both the `pgnum` property and the shipout hook, and order pages by the `.aux` order.
- **Two-column papers.** Use `posx` to find the column, take `x0` and `x1` from that column (`\columnwidth`, `\columnsep`), and split each region into per-column segments.
- **A start mark that lands at the bottom of a page** (its heading went to the next page) leaves a sliver. Check that start y is above `tbot + 30`. If it isn't, move the start to the next page's `CT`.
- Don't edit the user's uploaded files; work on copies. Keep the paper's layout byte-for-byte identical apart from the background boxes.
- **Local TeX Live older than arXiv's.** `00README.json` gives arXiv's TeX Live year. An older local install can change text without changing the page count: in one case, the `acronym` package's `\Acp` at the start of a sentence came out lowercase. After compiling, diff the per-page `pdftotext` output against the arXiv PDF (ignoring the arXiv side stamp on page 1), and report any difference to the user rather than editing the source.
- **A typed section unit isn't always a heading.** Empirical papers often use run-in `\paragraph{…}` for their units (for example "Training objective."). Box from those marks (see 3b), not from the enclosing subsection, or the box will swallow unrelated paragraphs.
- **Graph edges into the result node.** With `|-` from a node directly above, the arrow can run inside the result box. Use `(n) -- (n|-r.north)` for nodes above it, and route side nodes into `r.east` or `r.west`.

## Fallback: no TeX source, or it won't compile

Find the region starts and ends in the original PDF with `pdftotext -bbox` (heading words and end-of-sentence words; convert bp from the top to coordinates from the bottom). Then build an overlay with `pdfpages`:

```latex
\includepdf[pages=N,pagecommand={\begin{tikzpicture}[remember picture,overlay,x=1bp,y=1bp] ... \end{tikzpicture}}]{paper.pdf}
```

In the same coordinates, draw the same boxes with fill opacity at most 0.15, since they now sit on top of the text. Internal links in the paper are lost this way, so tell the user. Everything else (guide, merge, page labels) is unchanged.