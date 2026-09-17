# Voice of Salesforce — Incentive Compensation Sentiment Analysis

**A Power BI case study:** end-to-end analysis of a 1,649-respondent sales-force survey — from messy raw Excel exports to a 4-page interactive report with year-over-year trending, peer benchmarking, and demographic breakouts.



---

## Why this project

Most portfolio dashboards start from a clean Kaggle CSV. This one didn't — it started as a real-world messy dataset: inconsistent free-text fields, a survey structure that changed across years, a peer-benchmark file with a completely different shape than the internal data, and several genuine data-integrity bugs that only surfaced once real visuals were built on top of them. The value of this project isn't just the finished dashboard — it's the trail of decisions and fixes that got it there, documented below.

## The business question

Company X runs an annual "Voice of Salesforce" survey measuring how the sales force feels about its incentive compensation (IC) plan. Leadership needed to know:

1. Who took the survey (demographic profile)?
2. What's the overall sentiment, and which specific plan elements are weakest?
3. Does sentiment differ between managers and reps, or across therapeutic areas?
4. Is 2024 better or worse than prior years — and how does it compare to industry peers?
5. What do open-text responses say about how the plan *should* be designed?

---

## Data architecture

The dataset arrived as several disconnected sources: current-year raw survey responses, a question-to-category mapping sheet, and three separate benchmark files (this year's peer companies, and Company X's own 2022 and 2023 results) — each with a **different column structure** from the others.

Rather than forcing everything into one flat table, the model uses **two fact tables sharing one dimension table**, bridged through two different keys:

```
                    ┌─────────────────┐
                    │  Question_Map     │   (dimension)
                    │  Question_code    │
                    │  Statement         │
                    │  Theme              │
                    └─────────┬─────────┘
                    Question_code│  │Statement
                    ┌───────────┘  └───────────┐
        ┌───────────▼──────────┐   ┌───────────▼──────────┐
        │ Fact_Sentiment_Long   │   │ Fact_Benchmarks        │
        │ (2024, respondent-    │   │ (Peer / 2022 / 2023,   │
        │  level, unpivoted)    │   │  pre-aggregated,        │
        │                        │   │  unpivoted + appended)  │
        └────────────────────────┘   └──────────────────────┘

        Fact_Verbatims (open-text & plan-design picks — 
        intentionally NOT related to Question_Map, since 
        these questions aren't part of the 1–7 scoring system)
```

**Why two fact tables instead of one:** 2024 data is respondent-level (one row per person per question); the benchmark years are pre-aggregated segment averages with no individual respondents at all. Forcing them into a single table would mean inventing fake respondent rows for years where none exist. Two fact tables sharing one dimension is the standard pattern for exactly this situation — and it means `Question_Map[Theme]` can filter *both* fact tables simultaneously in any visual, even though they're joined on entirely different keys (`Question_code` vs. `Statement`).

---

## Data cleaning highlights

A few of the more interesting problems solved along the way:

- **A free-text tenure field with ~150 distinct spellings.** `Exp_in_domain` contained everything from `"20 years"` to `"Less than 1 year"` to three cells that Excel had silently auto-converted into dates (`1/2/2024` really meant "1–2 years"). Cleaned via a cascading `SUBSTITUTE()` formula in Excel that handled ~98% of entries automatically, with the remaining ~20 true edge cases (`"A lot"`, `"DCS"`, blank-worthy junk) manually reviewed and either mapped or nulled — never guessed.
- **The peer-benchmark categories didn't match my first theming attempt.** I initially grouped the 31 survey statements into 5 themes of my own design. On closer inspection, the `Peer Set Data` sheet already came pre-grouped into 4 named sections with subtotal rows (Manager Questions, Reporting & Communication, Business Rules, Motivation & Focus). Re-mapping every question to match the benchmark's own categories — rather than my invented ones — was what made the year-over-year and peer-comparison joins possible at all.
- **2023's benchmark data had no company-wide "Overall" row**, only 5 therapeutic-area-level "Overall" segments — while 2022 and the peer file both had a proper company-wide figure. Solved by approximating 2023's overall as the unweighted average of its 5 TA-level segments, explicitly flagged as an approximation everywhere it's used (never silently presented as exact).
- **4 survey items had no peer-benchmark equivalent at all** (a proprietary reporting-tool question, a supply-chain-specific question, and two "Impact Measures" items unique to Company X). Rather than force them into the 4 benchmarked themes, each got its own individual theme label — so they're reported on honestly rather than either hidden or misleadingly averaged into categories they don't belong to.

---

## DAX highlights

A few measures that solved genuinely non-trivial problems, not just simple aggregations:

**Ranking that ignores unrelated question types.** The dimension table includes 4 non-scored, multiple-choice questions (plan-design preferences) alongside the 31 real 1–7 scored statements. A naive `RANKX` would let those blank-scored rows silently occupy the top ranks (DAX treats blank as 0), pushing the *real* weakest statements out of a "bottom 3" filter entirely:

```dax
Statement Rank (Weakest) =
RANKX(
    FILTER(ALL(Question_Map[Statement]), NOT ISBLANK([Avg Score])),
    [Avg Score], , ASC
)
```

**Cross-fact-table comparison, despite the two tables never being directly related.** Because both fact tables filter through the same `Question_Map[Theme]` relationship, a measure can subtract values computed from *entirely different tables* and it just works:

```dax
Gap vs Peer = [Avg Score] - [Peer Overall Score]
YoY Change  = [Avg Score] - [Score 2022]
```

**Handling the 2023 structural gap** (no company-wide Overall segment) directly in DAX rather than faking it upstream:

```dax
Score 2023 =
CALCULATE(
    [Benchmark Score],
    Fact_Benchmarks[Year] = "2023",
    Fact_Benchmarks[Segment] IN {
        "Diabetes Overall", "Obesity Overall", "CV Overall",
        "RBD Overall", "Educators Overall"
    }
)
```

**Correct distinct-respondent counting** on a table where each person has up to 31 rows (one per question) — every headcount measure uses `DISTINCTCOUNT`/`AVERAGEX(VALUES(...))` rather than a plain row count, which would otherwise overcount by ~20–30x:

```dax
Total Respondents = DISTINCTCOUNT(Fact_Sentiment_Long[Unique Identifier])
```

---

## Report structure (4 pages)

| Page | Contents |
|---|---|
| **1. Demographics** | Respondent counts by therapeutic area, role level split, tenure distribution, with interactive tile-style filter buttons |
| **2. Sentiment Overview** | Average score by theme, %positive/neutral/negative breakdown, weakest & strongest individual statements |
| **3. Manager vs. Rep & TA Breakdown** | Grouped bar comparing manager/rep sentiment by theme, plus a color-scaled matrix of every therapeutic area × theme combination |
| **4. Trend & Peer Benchmark** | 2024 vs. peer-industry comparison, and a 4-year (2022/2023/Peer/2024) trend view per theme |


---

## Key findings

- **2024 sentiment declined on every single peer-benchmarked theme** — not just versus the industry, but versus Company X's *own* 2022 and 2023 results. This is a genuine trend, not one bad theme dragging an average down.
- **Manager Questions dropped the most**: −1.07 points vs. 2022, and 0.74 points below this year's peer benchmark — the clearest single priority in the data.
- **The two lowest-scoring statements in the entire survey are managers rating their own team's buy-in** ("the people I manage are motivated by the plan," "...believe the plan is fair") — not managers rating their own experience. Worth a targeted follow-up before assuming the root cause.
- **Reps overwhelmingly want a blended plan design** — in open-text responses, "a combination" was the most common answer for both what the plan should be based on and what metric it should use, ahead of every single-factor option.

---

## Challenges worth mentioning

- Debugged a recurring "sorts alphabetically instead of chronologically" bug across multiple visuals — root cause was a text column (`"0-2 yrs"`, `"20+ yrs"`) needing an explicit numeric `Sort by Column`, which required tracking down a genuine measure/table-reference mismatch before it would apply correctly.
- Diagnosed a data-type mismatch causing `AVERAGE()`/`AVERAGEX()` to fail silently on columns that *looked* numeric in the UI but were still typed as Text underneath — a good reminder to explicitly verify column types rather than trust visual formatting.
- Built (and then corrected) an initial theme-grouping scheme by comparing my own logic against the benchmark file's actual category structure, rather than assuming my first pass was right.

---

## Files in this repo

- `Final_document.pbix` — the full Power BI file (data model, all DAX measures, 4 report pages)
- `screenshots/` — page-by-page images for anyone without Power BI Desktop installed
- `README.md` — this document
