# NYC Traffic Safety Intelligence Dashboard

**Course:** DSPC A6000 — Advanced Computing for Policy  
**Team:** sparkly-pickle · Yiran Ge · Yizi Qu  
**Live Dashboard:** [sparkly-pickle.streamlit.app](https://sparkly-pickle-vluncnqsyee7xht3my7b7m.streamlit.app/)

---

## The Research Proposal

NYC sees tens of thousands of crashes every year. The instinct is to treat this as one problem with one solution — more enforcement, more cameras, citywide campaigns. But the data tells a more complicated story.

Our original proposal asked whether crash patterns vary by borough. After feedback that the framing was too descriptive, we sharpened it into three hypothesis-driven research questions:

- **RQ1:** Do crash *causes* vary by borough — and if so, does a uniform citywide enforcement strategy make sense?
- **RQ2:** Do crash causes look different at night versus during the day — are these structurally different problems requiring different tools?
- **RQ3:** Which boroughs are most dangerous for pedestrians specifically — and is that the same as where crashes are most common?

Each question was chosen because the answer has a direct policy implication, not just a descriptive finding.

---

## Project Evolution

| Stage | What Changed |
|-------|-------------|
| **Initial idea** | Visualize crash trends with borough-level bar charts, static CSV |
| **After proposal feedback** | "Show charts" → "Test hypotheses." Added 3 formal RQs with policy relevance |
| **During build** | Merged two datasets (Person + Crash tables); replaced static CSV with live BigQuery pipeline; sidebar filters made every chart interactive |
| **Final product** | 4-module policy story: big picture → who bears risk → where & when → policy levers |

The biggest shift was moving from *displaying data* to *guiding a policy argument*. The four-module structure isn't arbitrary — each module answers a prerequisite question before the next one becomes meaningful.

---

## Code

The codebase has three main concerns: getting data into a reliable database, querying it dynamically based on user filters, and keeping performance and consistency across all four pages.

### Data Pipeline

The pipeline runs automatically every day via GitHub Actions. It pulls crash-level and person-level records from the NYC Open Data API, merges them on `collision_id` to connect *cause* with *outcome*, standardizes the columns, and writes the result to Google BigQuery. The dashboard never hits the raw API at runtime — it always queries a pre-processed table.

The most important fix in the pipeline was handling the time field. The API returns `crash_time` as a full datetime object, not a simple time string. If you slice it like a string, you get the year instead of the hour. We had to extract just the time component and recombine it with the correct date:

```python
def merge_date_and_time(df):
    date_only = pd.to_datetime(df["crash_date"], errors="coerce").dt.date
    time_only = pd.to_datetime(df["crash_time"], errors="coerce").dt.time
    ...
    df["crash_date"] = temp.apply(combine, axis=1)
    return df
```

Without this fix, all the time-of-day analysis in Module 3 would be wrong.

### Dynamic Query System

Pages don't load the full dataset — they send a targeted SQL query that returns only the slice matching the current filter selection. BigQuery handles the heavy aggregation; Streamlit handles the presentation. The user's filter choices (borough, date range, time of day) get translated directly into SQL conditions, so one query function supports the full range of interactions without writing a separate version for every combination.

### Filters, Parallel Queries & Cache

All filter logic lives in one shared module imported by every page. One change updates all four modules at once, with no risk of filters behaving differently on different pages. To improve performance, independent data requests are fetched in parallel (wait time = slowest query, not the sum), and results are cached for one hour so repeated interactions return instantly.

---

## Key Findings

### Who bears the risk

Pedestrians make up only **3.3%** of people involved in crashes, but **57.6%** of traffic deaths — a disproportionality index of **17.5**. A policy focused on the average driver will not close this gap.

### Where & When

**RQ1 — Crash causes vary by borough.** Driver inattention dominates everywhere, but secondary causes diverge. A uniform citywide campaign misses local priorities.

![Crash causes by borough](rq1_borough_causes.png)

**RQ2 — Late-night crashes are a different problem.** Unsafe speed accounts for 10% of late-night crashes vs. 5.6% during the day — 1.8× higher. Alcohol involvement also rises. Daytime enforcement tools are the wrong fit after 10pm.

![Daytime vs. late-night crash causes](rq2_day_night.png)

**RQ3 — Pedestrian risk and crash volume rank boroughs differently.** Brooklyn leads in total crashes; Queens and the Bronx lead in pedestrian death share. Funding pedestrian safety by crash volume sends money to the wrong places.

![Borough ranking: total crashes vs. pedestrian death share](rq3_pedestrian_ranking.png)

---

## What I Learned

The most lasting lesson is the difference between *showing data* and *using data to support a decision*.

Early in the project, stopping at a bar chart of crash counts by borough would have felt like analysis. Building this dashboard pushed further — toward fatality *rates* instead of raw counts, pedestrian death *share* instead of total volume, and a page structure that guides the user from a broad observation to a specific policy question. That progression doesn't happen automatically; it has to be designed.

This project also clarified what it means to transform data into a policy tool. The technical work — pipeline, database, filters — is only half of it. The other half is deciding what question each visualization is actually answering, and whether that question is one a policymaker can act on. A chart of crash counts is interesting. A chart showing that the borough receiving the most pedestrian safety funding has the lowest pedestrian death share is *actionable*.

On the technical side, connecting a live database to an interactive interface taught me what separates something *functional* from something *usable*. Caching, parallel queries, and shared filter logic aren't features — they're the difference between a tool someone will actually use and one they'll abandon after the first slow load.

---

## Technical Stack

Python · Altair · Streamlit · Google BigQuery · GitHub Actions · NYC Open Data API
