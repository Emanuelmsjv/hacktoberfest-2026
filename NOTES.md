# Notes

This project is something I genuinely like and want to keep making better and more usable in everyday life. The goal is to make it something I'd actually use day-to-day.

## Roadmap (rough ideas)

1. **Phone -> LAN import** - Allow importing receipts from phone (e.g., over LAN/HTTP share) so snapping bills on the go feeds straight in without needing to copy files to laptop.
2. **Monthly digest** - Generate a simple monthly summary (total spend, top vendors, anomalies) as a single markdown report or printable page.
3. **Local LLM Qs** - Ask simple questions ("How much did I spend on X in September?") running entirely offline with a local LLM; answers grounded in `data/ledger.json`.
4. **Auto-categorization** - Suggest categories from vendor/line items; remember corrections locally.
5. **Review loop** - A tiny "needs review" list for low-confidence fields (OCR fallback cases) to steadily improve correctness without guessing.
6. **Recurring charges** - Detect likely subscriptions/recurring payments and surface them explicitly.
7. **Export to common tools** - Optional exports (Tally/Zoho) that are opt-in and never upload anything.

Principle: offline, no API keys, no cloud, and *"null not guess"* stays the north star.