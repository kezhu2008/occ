# Ledger Charter

> Owned by founder + CEO. The **business vision only** — why the company exists and what success means. No technical design here: types, storage formats, frameworks, and invariants live in `architecture.md` / `data-model.md` (architect-owned). Founder confirms before recruiting.

## Business intent

Ledger is a portfolio and Australian-tax tool for self-directed investors who hold through Interactive Brokers. It exists to turn a year of brokerage activity — every trade, dividend, option, corporate action, currency conversion and transfer — into the answers an investor actually needs at tax time and through the year: what do I own, what is it worth, what is my capital-gains and income tax position per taxpayer, and what could I do before 30 June to improve it. The market it serves is underserved today: Sharesight and its peers treat Interactive Brokers as a second-class import and are weak on options, multi-currency, and the parcel-level Australian-tax detail that decides the number. Ledger is built first-class around the IBKR feed and around getting the tax right to the cent — including the cases (options, foreign currency, corporate actions, missing-history transfers) that the generic tools get silently wrong. The founder is user-zero: the product is proven on a real portfolio before it is offered to anyone else.

## User

A self-directed Australian investor — a "value investor" who researches and holds, trades through Interactive Brokers across one or more accounts and tax entities (their own name, an SMSF, a company), and holds shares and options across multiple currencies. Their job-to-be-done: **know my true position and my correct Australian tax, without hand-rebuilding it in a spreadsheet, and trust the number enough to file on it.** They will not accept a tool that is approximately right; a wrong capital-gains figure is worse than no figure. They want to be told honestly when the data is incomplete (e.g. a transferred-in holding with no cost history) rather than handed a confidently-wrong answer.

## Success definition

- **Founder runs their real portfolio on it.** The founder connects their actual Interactive Brokers account(s), gets a complete per-entity picture — positions, valuations, capital gains, franking, foreign income, currency gains, end-of-year tax summary, and the optimisation suite — and uses it to file and to make decisions. This is the first and decisive proof of success.
- **The numbers are trustworthy to the cent and reproducible.** The tax figures are correct against a hand-derived check across each taxpayer type and each financial year, and the same inputs always produce the same outputs — the investor (or their accountant) can re-run it and get an identical answer.
- **The hard cases are handled, not hidden.** Options, multi-currency conversions, corporate actions, foreign tax, and incomplete cost history are all surfaced and resolved — data gaps are flagged and fillable rather than silently producing a wrong gain.
- **It runs unattended on the founder's behalf (W6).** The backend keeps the founder's portfolio in sync on a schedule and on demand without the phone being open, and produces tax numbers identical to the on-device build — so "scale beyond myself" begins with the founder's own always-on backend.
- **It is offered to others.** Beyond personal use, another investor can install it, connect their own account, and get their own isolated, correct per-entity result — the point at which Ledger is a product, not a personal tool.

## Business non-negotiables

> Constraints that are product/market/values decisions, not engineering choices.

- **Tax numbers are correct to the cent.** This is the core trust promise. An approximation that is "close" is a failure; the figure must withstand a hand check and an accountant's scrutiny.
- **Same inputs, same answer, every time.** The investor and their accountant must be able to re-run the tool and reproduce the exact tax result — auditable, not a moving target.
- **Honest about incomplete data.** Where the brokerage feed cannot know the full history (e.g. a transferred-in holding), Ledger flags the gap and lets the investor fill it — it never emits a silently-wrong capital gain.
- **Each tax entity is treated as its own taxpayer.** An individual, an SMSF and a company are separate filers with separate rules; their results are never blended.
- **The investor's brokerage credentials and financial data are handled as sensitive.** During personal use they stay on the investor's own device; once the product is hosted for others, each user's data and credentials are isolated and protected, and the handling is disclosed. A user's data is never visible to another user.
- **Australian-investor scope is deliberate.** Ledger targets Australian tax (capital-gains discount, franking, foreign-income offset, currency gains) reported in Australian dollars; this focus is a product choice, not a limitation to apologise for.
- **It ships to real use.** The bar is a tool the founder relies on daily and, ultimately, an App Store product — not a demo.

## Scope & horizon

- **Shape:** single product, delivered as a multi-wave suite (a personal tool that grows into a hosted product).
- **In scope:** portfolio and positions (shares **and** options), watchlist, valuations and daily performance, cash and currency balances, the full Australian-tax suite (capital gains, franking, foreign income, currency gains, corporate actions, end-of-year summary), tax optimisation, and an insight/report catalog with export — first on the investor's own device, then synced and served from a backend.
- **The backend lift (W6) is in scope and is a lift, not a rebuild:** the proven calculation engine is kept and moved behind the backend so the founder's portfolio syncs on a schedule and on demand without the phone open, with tax results identical to on-device — done first as the founder's own single-tenant always-on backend, then, if and only if the product is to serve others, extended to a multi-user hosted product with per-user isolation and sign-in.
- **North star (deferred, beyond this plan):** research and valuation tooling for the same investor.
- **Rough horizon:** the personal-use product through daily iPhone use, then the daily-performance/cash and foreign-currency-correctness corrective work, then the backend lift (W6), then the optional multi-user product.

## Founder availability & autonomy

- **Low-touch within a wave; founder crosses the boundary between waves.** The company runs each wave autonomously and reports; the founder is asked only at the points that genuinely need them — granting credentials and spend, approving a path that changes cost or risk, doing the manual external steps, and re-testing the result on their real portfolio. The escalation bar is calibrated accordingly: decide and proceed inside a wave, escalate the boundary crossings and any choice that changes scope, spend, or the trust promises above.

## Stack (one line, pointer only)

- A native iOS app for the investor over a portable calculation core, lifting in W6 to a cloud backend that syncs and computes on the investor's behalf and serves the app — detailed design lives in `architecture.md` / `system-design.md`.
