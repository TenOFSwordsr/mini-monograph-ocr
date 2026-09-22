# Mini Monograph OCR / Text Extraction

Converts a 617-page Persian drug reference, «مینی مونوگراف دارویی» (Mini Monograph), into clean
Markdown. The PDF has an embedded text layer, but in the wrong reading order for an RTL two-column
layout, so the pipeline reconstructs visual lines from block geometry, emits columns right-to-left,
anchors every text run under its printed box caption (برند معروف، اشکال دارویی، اندیکاسیون،
رده بارداری، رده شیردهی، عوارض و تداخلات، نکات، منع مصرف، Black Box Warning), and keeps Tesseract
output as a cross-check. Fifteen successive `build_md*.py` revisions sharpened the parser in one
sitting; `build_md14.py` is the final one.

**Suggested repo name:** `mini-monograph-ocr`
**Stack:** Python 3, PyMuPDF (`pymupdf`), Tesseract OCR CLI with `fas+eng`, `ProcessPoolExecutor`
**Status:** finished
**Last modified:** 2026-09-04

## What it does

Stages, in run order:

- `render.py` - rasterizes all 617 pages to `pages/page_NNNN.png` at 3.0x zoom, 8 processes.
- `ocr_pages.py` - Tesseract per page (`-l fas+eng --psm 4 --dpi 300`) → `ocr/page_NNNN.txt`,
  12 workers, skips non-empty existing files so it resumes.
- `ocr_tsv.py` - same pass in TSV mode → `tsv/page_NNNN.tsv` for word/box coordinates.
- `extract.py` - alternative path: reads the native text layer via `page.get_text('blocks')`,
  clusters blocks into lines by y-centre (11 pt tolerance), sorts each line right-to-left, and
  writes `reflow/page_NNNN.txt`. Character-exact where the text layer is trustworthy.
- `proto.py` - prototype reflow that also splits a line into column groups on a 25 pt x-gap and
  joins them with `||`; used to eyeball pages 540 and 137.
- `build_md.py` … `build_md14.py` - the Markdown assembler. Each revision adds parsing accuracy:
  caption→header mapping, the two-column `||` separator, Black Box folded into «منع مصرف»,
  front-matter pages (title, how-to, TOC, author lists) passed through unparsed, `load_ocr_words()`
  used to cross-check the text layer, and a final consecutive-line dedup. `build_md14.py`
  (470 lines) is the version to use.
- Indexes produced for navigation: `card_names.txt` (576 `<page>\t<English drug>` rows) and
  `drug_map.json` (191 drug names → page numbers). `lalef_map.json` is a 51-entry table of
  mis-extracted Persian word forms → correct spellings.

## Layout

```
render.py ocr_pages.py ocr_tsv.py     rasterize + Tesseract stages
extract.py proto.py                   text-layer reflow (geometry-based, RTL aware)
build_md.py … build_md14.py           Markdown assembler, 15 revisions (use build_md14)
card_names.txt drug_map.json          page/drug indexes
lalef_map.json                        Persian orthography correction table
pages/ ocr/ tsv/ reflow/              generated intermediates (617 files each)
md/                                   empty - reserved, unused
```

## Running it

Requires the source PDF and Tesseract at the paths coded in the scripts:

```bash
python render.py
python ocr_pages.py
python ocr_tsv.py
python extract.py
python build_md14.py      # -> Downloads/Mini_Monograph_OCR.md
```

## Notes

- **Both the input and the output live outside this folder.** `PDF` is hardcoded to
  `C:\Users\Administrator\Downloads\compressed_Mini Monograph completed (1)_compressed-compressed.pdf`
  and `OUT_MD` to `C:\Users\Administrator\Downloads\Mini_Monograph_OCR.md`, as are the
  `pages/`/`ocr/`/`tsv/` paths inside the OCR scripts. Nothing runs on another machine without
  editing those constants, and the repository is incomplete without the generated Markdown.
- `pages/`, `ocr/`, `tsv/` and `reflow/` are 617-file generated caches (~2,468 files total) -
  ignore them in Git; keep only the scripts and the two JSON/TXT indexes.
- The 14 intermediate `build_md*.py` revisions are kept as working history, not as variants;
  delete 1–13 if you want a clean tree.
- The book is third-party clinical content. Publishing the extracted Markdown means publishing
  someone else's copyrighted reference; the extraction code is separable and safe to share.
- Companion work: `Downloads/pdf-extract` transcribes handwritten Persian pharmacology notes to
  the same monograph field structure from a different source.
