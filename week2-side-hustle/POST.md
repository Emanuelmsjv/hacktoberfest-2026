*This is a submission for the [Hacktoberfest Open-Source AI Challenge Week 2: Side Hustle](https://dev.to/challenges/hacktoberfest-week2-2026-10-12)*

## What I Built

**LedgerLens** turns a folder of receipt photos into an accounting CSV, then finds the bills you already paid.

Point it at a folder of `.jpg`/`.png` receipts. It runs OCR on your own machine, extracts the vendor, date, invoice number, line items and total from each one, writes one clean row per receipt to `data/ledger.csv`, and flags the things a human skimming a spreadsheet misses: a bill entered twice, a price that jumped, a total far outside what that shop normally charges, or items that do not add up.

It is for the freelancer or small shop owner who keeps a shoebox of receipt photos and no useful record of them. The information is all there, nobody has time to type it in, and the cloud services that would do it charge per page and upload every receipt to a server you do not control.

That last part is the whole point. A shop's receipts carry client names, GST numbers, addresses and staff details. **LedgerLens never uploads an image anywhere.** Tesseract runs as WebAssembly inside Node, so there is no Python, no native toolchain, no API key, and no per-image bill. After a first run that downloads about 11 MB of English language data, it works with the network switched off.

## Demo

```text
$ npm run pipeline

         78.0 conf          0.5s  receipt-01.png
         82.0 conf          0.2s  receipt-02.png
         78.0 conf          0.4s  receipt-03-duplicate.png
         76.0 conf          0.3s  receipt-04-stationery.png

4 image(s) -> data/raw
    ANNAMAIYA PROVISION STORES       2026-10-12    1451.34  ok  via total
    ANNAMAIYA PROVISION STORES       2026-10-14     575.52  ok  via total
    ANNAMAIYA PROVISION STORES       2026-10-12    1451.34  ok  via total
    SARASWATHI STATIONERY            2026-10-15     649.64  ok  via total

4 receipt(s), 4 with a resolved total -> data/ledger.csv
  receipts : 4
  findings : 1

  [HIGH] duplicate_invoice
      Invoice 2291 from ANNAMAIYA PROVISION STORES appears 2 times at an
      identical 1451.34. Likely a double scan or a double payment.
```

Those are the synthetic samples the repo ships. On real photographs it reads the paper and hands back one row per bill.

## Code

{% embed https://github.com/Emanuelmsjv/ledgerlens %}

## How I Built It

Plain Node.js, one dependency (`tesseract.js`), no build step. The pipeline is four small scripts:

| script | what it does |
| :--- | :--- |
| `ocr.mjs` | images to text, with rotation recovery |
| `parse.mjs` | text to one CSV row per receipt, plus a self-check |
| `analyze.mjs` | flags duplicates, price jumps, outliers, bad arithmetic |
| `make-fixtures.mjs` | redacts real OCR output into safe test fixtures |

The design rule everything else follows from: **if a field cannot be read confidently, it returns `null`, never a guess.** In a ledger, a blank cell is a nuisance; a confidently wrong figure is a false expense that gets filed and never questioned again.

### The bug that rule comes from

My first parser took the *last* line containing "Total". On one real bill, three lines contained the word. It returned **459.00** where the bill said **4600.00** — with no warning, no low confidence, nothing.

I did not trust my memory of how bad that was, so I rebuilt the original logic and ran it against every real bill I had ground truth for. It was worse than I expected:

| bill | true total | original logic | shipped |
| :-- | --: | --: | --: |
| railway ticket | 180 | *(nothing)* | 180.00 |
| bakery | 593 | **102.00** (wrong) | 593.00 |
| bakery (other photo) | 593 | **102.00** (wrong) | 593.00 |
| grocery | 88 | 88.00 | 88.00 |
| credit sale | 4600 | **459.00** (wrong) | 4600.00 |

**Original: 1 right, 3 wrong, 1 abstained. Shipped: 5 right, 0 wrong.**

The fix was not a better regex. It was label priority: prefer a `NetAmount` or balance line, never take a tender or "savings" figure, and when two candidates disagree, abstain. Measured, not claimed.

### What the benchmark says

Two suites, reported separately and never averaged together:

| | synthetic | real |
| :-- | --: | --: |
| cases | 7 | 8 photos, 4 distinct bills |
| ground truth | fixed when written | read off the paper by eye |
| totals correct | **7 / 7** | **7 / 8** |
| **wrong** | **0** | **0** |
| wrong-rate of produced totals | **0%** | **0%** |

**14 of 15 documents, and no wrong number anywhere.** The single miss is the system working: on one photograph it could not read the total cleanly, so it reported nothing. Coverage is the secondary metric; a wrong number is the only real failure.

The same rule caught a GSTIN that OCR had turned from `33ZZQPX1234F1Z2` into `332ZQPX1234F122` — a shape that is no longer a valid GSTIN. The parser returned `null` and left it for a human, instead of filing a mangled tax number on a real invoice.

The suite is covered by **68 checks** across three files (`30/30`, `12/12`, `26/26`), plus a 15-check privacy lint wired into CI.

### And the part I got wrong about privacy

Halfway through, I checked whether the real receipts had leaked into the public repo. The photos had not — `samples/real/` was gitignored from day one. But the **redacted fixtures** derived from them were incomplete: real shop names, a street address, a pincode and bill numbers were sitting in fixtures, a benchmark file, and even the CI workflow. Worse, the test suite depended on a gitignored file, so **the privacy guard in CI had never actually run** — it crashed before reaching the check.

I fixed it properly: rewrote the redaction rules, added a fixed-point lint that fails the build if a fixture is not idempotently redacted, made the tests hermetic so the guard runs on a clean clone, and purged the leaked content from git history. The public repo now contains only fictional vendors. The gate reports `15/15`.

I am including this because a tool whose whole pitch is "your data stays yours" has to hold itself to that, and mine did not at first.

## Why Does Open Innovation Matter?

For this project, open isn't a preference. It is the reason the thing works at all.

A closed receipt-OCR API bills per image. For a shop doing hundreds of small purchases a month, that is a recurring cost that eats the margin it is supposed to protect — and every receipt, each one a page of someone's financial life, gets uploaded to a server they do not own. **Running the model locally removes both the metered cost and the data exposure in one move.** That is the entire product.

It also means I can trust it. The extraction rules and the five detectors are plain code I can read, test and fix. When the "Total" bug turned up, I could rebuild the old logic and measure it against real bills the same afternoon — no black-box API to wait on, no behaviour changing under me. Everything runs offline, on CPU, with no key and no account. That combination — no metered cost, no third-party exposure, and rules you can audit — is what open innovation made possible here. I do not think the closed version of LedgerLens is worth buying, and I do not think that is a coincidence.

## My Agent Session

I built this end to end with an AI coding agent, including the measurement above and the privacy remediation. The session below is the slice where I stopped trusting a claim, rebuilt the original parser, and found the bug was worse than I had said.

{% agent_session 919 %}

## Prize Categories

I'm entering:

- **Overall** — a genuinely useful, fully offline side-hustle tool with measured correctness.
- **Entire** — the agent session above shows how it was built and where it went wrong.
