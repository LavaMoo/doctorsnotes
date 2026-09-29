# Visit Prep Brief

A concept prototype that turns a patient's messy list of worries into a one-page, prioritized brief to bring to a doctor's appointment.

> **Concept prototype. Not medical advice. Do not enter real personal health information.**
> This project uses fake data only and does not diagnose, treat, or recommend anything.

## What it does

1. Optional visit basics (visit type, conditions, medications, visit length)
2. A free-text "brain dump" of concerns
3. A review screen to edit, re-categorize, prioritize, and reorder concerns
4. A printable one-page brief, with copy-as-text

If the text contains urgent-symptom phrases, the app shows a banner telling the user to call their local emergency number.

## Run it

No build step and no dependencies. Open `index.html` in a browser.

To host it free on GitHub Pages: Settings, Pages, deploy from the `main` branch, root folder.

## How it works

- Single-file vanilla JS app; all data stays in the browser tab and is never sent anywhere.
- The concern parser is rule-based (keyword tables and simple heuristics near the top of the script), so it is easy to tune.
- The parser is a pure function (`parse`) and can be swapped for an LLM-backed version later.

## Docs

The full implementation spec is in [`docs/spec.md`](docs/spec.md).

## Not included yet

LLM-powered parsing, drag-and-drop reordering, saved drafts, multiple visits, voice input.

## License

MIT. See [`LICENSE`](LICENSE).
