# Scriptiq

A full-stack tool for translating manga into English. Upload a page or a whole chapter, and Scriptiq:

1. finds the speech bubbles
2. reads the Japanese
3. translates it with a vision LLM that can see the page
4. typesets the English back into the original art

Every step can be reviewed and corrected in a browser editor before export.

**Results:** 93% text-extraction accuracy across 1,000+ pages, tested with 2 translation teams.

**The source is private for now.** This page covers how it works and the problems that shaped it.

`Python` `FastAPI` `OpenCV` `EasyOCR` `manga-ocr` `Claude` `React` `Docker` `Fly.io`

---

## Why it's hard

Translating manga is more than OCR plus machine translation:

- **Japanese is written in narrow vertical columns.** English needs the whole width of the bubble, so the text box the OCR finds is the wrong shape to typeset into.
- **A bubble isn't a sentence.** Lines are short, speakers alternate, and jokes and names have to stay consistent across hundreds of pages. Translating one bubble at a time gives literal, flat dialogue that often mixes up who is talking.
- **Pages are noisy.** Page numbers, scanlation watermarks and sound effects all look like text to a detector.

## Pipeline

```
page image
  │
  ├─ detect     EasyOCR finds text fragments → merged into bubbles by gap and column
  │             alignment (scaled to page size) → page numbers and watermarks dropped
  │
  ├─ read       manga-ocr reads each bubble locally: vertical text, furigana, hand-lettered fonts
  │
  ├─ translate  a vision LLM sees the page with every bubble outlined and numbered,
  │             the OCR text, the series memory, and the last two pages of dialogue
  │             → returns a speaker and a translation per bubble (structured JSON)
  │             → updates the series memory
  │
  ├─ typeset    OpenCV grows each text box out to the bubble's real outline,
  │             clears the interior, and fits the English with auto-sized, wrapped text
  │
  └─ export     PNG per page, or a zip for a whole chapter
```

## Engineering highlights

**Fitting English to the bubble, not the text box.** Detection hugs the Japanese text, which is usually a tall, thin column. `bubble_shape` starts from the white background between the characters and flood-grows the light region until it hits the bubble's outline. It then fills the holes the characters left. The result is the bubble's real interior, which is where the English goes. Text that isn't inside a white bubble, such as captions on art, is drawn with an outline instead of a white box, so the artwork is never covered.

**Translating the page, not the bubble.** The model gets an image of the page with numbered outlines drawn on, so it can see who is speaking to whom. It must answer in a fixed JSON schema: a speaker, a translation and an optional note for each bubble. It also returns an updated **series memory** with characters (how they look and talk), a glossary, running jokes and a story summary. That memory is fed into the next page, so names and voices stay consistent across a whole series. You can view and edit the memory from the editor.

**Swappable engines.** Claude is the default. Groq runs the same page-aware approach on a cheaper, faster vision model. DeepL gives literal bubble-by-bubble translation for comparison. Detection, OCR and furigana all run locally, so only the translation step calls an API.

**Filtering noise.** Fragments smaller than about 2% of the page's longest side are dropped as stray marks. Small regions near the top or bottom edge are dropped as page numbers. After OCR, any region that reads as letters or digits with no Japanese at all, such as "771" or a watermark URL, is filtered out. All thresholds scale with page size, so they work on low- and high-resolution scans alike.

**Editor.** A three-panel React editor, with the page in the middle, an overlay of the detected bubbles, and the script beside it. You can add, move and resize bubble boxes, reassign speakers, rewrite lines, and brush out leftover art with a mask tool. A furigana mode adds hiragana readings next to the kanji instead of translating, for people learning Japanese.

**Library and chapters.** Pages autosave. Chapters can be built from a folder or zip upload, reordered, translated in one pass on the server, and downloaded as a zip.

**Hosting.** One Docker image serves the API, the built frontend and a sign-in page from a single address on Fly.io. Pages and series memory live on a persistent volume. The machine sleeps when idle and wakes on the next visit in about 30 seconds.

## Limitations

OCR is Japanese only for now. Korean (manhwa) and Chinese (manhua) need a different OCR model, and the non-dialogue filter assumes Japanese.

---

Built by [Prabu Ramanababu](https://github.com/prpr00007) · [LinkedIn](https://www.linkedin.com/in/praburamanababu) · happy to demo it or walk through the code.
