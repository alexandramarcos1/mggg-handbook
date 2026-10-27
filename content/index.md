---
title: MGGG Handbook
---

This is a start-here guide for new lab members. It explains, in plain language, the methods and concepts the lab uses to study how ranked-choice voting and different district maps affect proportional representation. It is built as a **two-week plan**: each day has a short reading and one task, and section 8 lays out both weeks.

## 1. What this lab does

We study how election rules and district maps shape **who gets represented**, using simulations. We cannot rerun a real election under different rules, so we **simulate** it.

**The question the lab asks:** *If a city changed how it elects its council (for example from single-member districts with plurality to multi-member districts with ranked-choice voting), would communities of voters get fairer representation? If so, does the ranked-choice voting (RCV) system succeed in proportionally representing community groups?*

Every project follows the same loop:

1. **Ask** a fairness question, such as “would a community group (bloc) holding 20% of voters win about 20% of seats under a different electoral system?” This is an important question: the lab operationalizes fairness as the share of seats a community holds in office (city council, parliament, school board) relative to its share of the voting-age population. This community population is what you will know as VAP.
2. **Model** the place. Each state and city has different parameters and variables, so we try to represent the place we want to study as faithfully as possible. We use information on demographics, district maps, historical turnout, voting patterns (cohesion and polarization), and the historical number of candidates.
3. **Simulate** many elections under each rule and map.
4. **Validate** the assumptions with people who know the place (community groups, local officials, researchers).
5. **Refine** the model and repeat.

Step 4 is what separates our work from a pure math exercise: a model is only as good as its assumptions, and local experts tell us how to improve those assumptions.

Steps 2 and 3 follow a predetermined pipeline, summarized as:

```
 Map data (census blocks or precincts (VTDs))
        │  GerryChain draws many possible district maps
        ▼
 District plans (e.g. 3 districts x 3 seats)
        │  turn demographics into voter blocs + turnout
        ▼
 Election settings (JSON)
        │  VoteKit generates synthetic but realistic ballots
        ▼
 Ballots ("preference profiles")
        │  VoteKit counts them under each voting rule (STV, plurality, IRV...)
        ▼
 Winners  ──►  Summaries, histograms, figures  ──►  Report
```

Two libraries do most of the work:

| Library        | Job                                    | Repo                               | Docs                                      |
| -------------- | -------------------------------------- | ---------------------------------- | ----------------------------------------- |
| **GerryChain** | Draws thousands of valid district maps | https://github.com/mggg/GerryChain | https://gerrychain.readthedocs.io         |
| **VoteKit**    | Makes ballots and runs elections       | https://github.com/mggg/VoteKit    | https://votekit.readthedocs.io/en/latest/ |

> **How to use this handbook.** Follow the plan in section 8: read a few sections each day, then do that day’s task. Week 1 covers this handbook; Week 2 is a project on a city of your choice.

## 2. Vocabulary

| Term                             | Plain meaning                                                                                                                                                   |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| VAP (voting-age population)      | People 18 and older; our basic measure of a group’s size                                                                                                        |
| Bloc                             | A group of voters who tend to vote alike (for example, by race or ethnicity)                                                                                    |
| Slate                            | The candidates a bloc fields or prefers                                                                                                                         |
| Cohesion                         | How strongly a bloc votes for its own slate                                                                                                                     |
| Crossover                        | The share of a bloc’s votes that goes to another bloc’s slate                                                                                                   |
| Turnout rate                     | The share of a bloc’s VAP that actually votes, relative to other blocs                                                                                          |
| District, seat and magnitude (m) | A district is an area; a seat is one position; the magnitude (m) is how many seats a district elects                                                            |
| Districting notation (D × m)     | “3 × 3” means 3 districts with 3 seats each (9 seats); “9 × 1” means 9 single-seat districts; “1 × 6” means one at-large district (the whole city) with 6 seats |
| Single-winner vs. multi-winner   | One seat per district vs. several seats per district                                                                                                            |
| Proportional representation      | A group’s share of seats roughly matches its share of voters                                                                                                    |
| Vote dilution                    | Lines or rules that make a group’s votes count for less, by splitting it across districts or packing it into one                                                |
| Ensemble of maps                 | Thousands of random, legal district plans, used to see what is typical rather than what one map does                                                            |
| Charter                          | A city’s local constitution; changing election rules there often needs a voter-approved amendment                                                               |
| Ranked ballot                    | Voters rank candidates in order of preference instead of choosing one                                                                                           |
| Preference profile               | The full set of ballots cast in an election                                                                                                                     |
| Polarization                     | Blocs rank candidates very differently from one another                                                                                                         |

## 3. The math toolkit

You need arithmetic, percentages and a feel for probability. Each idea below has a formula, a tiny example and the question it answers.

### 3.1 Group share and proportional share of seats

How big is a group, and how many seats would it hold if seats matched its size?

$$
\text{share}_g = \frac{\text{VAP}_g}{\text{total VAP}},
\qquad \text{proportional seats}_g = \text{share}_g \times S
$$

Example: a group with 20% of VAP in a 9-seat council has 0.20 × 9 = 1.8 proportional seats. If it holds 1 seat, its gap (seats held minus proportional seats) is −0.8 seats. From here on we say *proportional seats* rather than *fair seats*, because proportionality is something we can measure, while fairness is a judgment.

### 3.2 Relative turnout adjustment

Only people who vote matter, so we shrink each bloc by its turnout rate t.

$$
\text{effective voters}_g = \text{VAP}_g \times t_g,
\qquad \text{effective share}_g = \frac{\text{VAP}_g \times t_g}{\sum_h \text{VAP}_h \times t_h}
$$

Example: blocs of 500,000 (t = 0.8) and 200,000 (t = 0.4) give 400,000 and 80,000 effective voters. The second bloc’s share drops from 28.6% of VAP to 16.7% of voters.

This parameter is important, so it is worth being precise about it. VoteKit uses a **relative** turnout: what matters is the ratio between blocs, not the absolute rates. Bloc 2 has a turnout of 0.4, but compared with bloc 1 (0.8) its relative turnout is 0.4 / 0.8 = 50%. Setting the turnouts to 1.0 and 0.5 gives 500,000 and 100,000 effective voters, and the same 16.7% share. Scaling every bloc’s turnout by the same factor never changes the result.

Turnout rates come from historical data, such as the Census Bureau’s Current Population Survey (CPS) voting supplement.
*Source: U.S. Census Bureau, Current Population Survey Public-Use Data.*

### 3.3 Cohesion and crossover

We store voting patterns in a table: each **row** is a voting bloc, each **column** is a slate, and each row adds to 1.

| Voters ↓ / Slate → | Slate A | Slate B |
| --- | --- | --- |
| Bloc A | 0.85 | 0.15 |
| Bloc B | 0.10 | 0.90 |

Bloc A voters give 85% of their support to slate A (cohesion) and 15% to slate B (crossover). The table need not be symmetric. Check every row sums to 1 before running anything.

### 3.4 Splitting support inside a slate (Dirichlet)

If a bloc gives 80% of its support to a slate of k candidates, how is that 80% divided? We draw shares from a Dirichlet distribution with a parameter α.

- **Small α (under 1):** support concentrates on one candidate, a “rockstar.”
- **α = 1 (neutral):** every possible way of splitting the support is equally likely. The draw is uniform over all splits: random, with no pull toward a rockstar and no pull toward an even split. Any single bloc can end up lopsided or balanced, and across many voters there is no systematic favorite. This is the default in our baseline configs.
- **Large α (2 or more):** support spreads almost evenly.

Each candidate’s average share is 1/k whatever α is; only the spread changes.

$$
\text{Var}(\text{one share}) = \frac{\frac{1}{k}\left(1-\frac{1}{k}\right)}{k\alpha + 1}
$$

Example, k = 3: α = 0.5 gives variance 0.089 (standard deviation about 0.30); α = 1 gives 0.056 (about 0.24); α = 2 gives 0.032 (about 0.18).

Reading it: with α = 0.5 most voters have a clear favorite, and *which* candidate is the favorite changes from voter to voter. With α = 2 most voters spread their support fairly evenly, so the voters look alike. With α = 1 you get a mix.

### 3.5 How many candidates run (geometric distribution)

We draw a slate’s pool size from a geometric distribution: keep flipping a coin with success chance p; the number of flips until the first success is the pool size.

$$
P(K = k) = (1-p)^{k-1}\,p, \qquad E[K] = \frac{1}{p}
$$

Example: p = 0.5 gives an average pool of 2 candidates; p = 0.25 gives 4. We usually scale the pool by the bloc’s share of voters in each district, which assumes small groups field fewer candidates. That assumption is worth checking with local experts.

### 3.6 Winning thresholds (Droop quota)

In multi-winner ranked-choice elections (STV), a candidate wins on reaching the quota.

$$
\text{quota} = \left\lfloor \frac{\text{votes}}{\text{seats}+1} \right\rfloor + 1
$$

With 3 seats the quota is just over 25% of votes; with 1 seat it is a majority. This is why a group near 25% can win a seat in a 3-seat district but a dispersed group of the same size cannot in 9 single-seat districts. This is a quick way to estimate whether a bloc will win a seat based on their VAP proportions in a region. However, this analysis doesn’t incorporate the turnout, cohesion and Dirichlet parameters, which add randomness to the simulation.

### 3.7 The scorecard

For each scenario we run many simulations and report two numbers per bloc: **expected seats** (the average) and the **probability of winning at least x seats**. We compare expected seats with proportional seats from 3.1. A negative gap means under-representation.

## 4. Election systems at a glance

We compare systems on how each turns the same ballots into seats. “D × m” means D districts with m seats each (see section 2).

| System | How voters vote | How winners are chosen | Tends to help |
| --- | --- | --- | --- |
| Plurality (single-winner) | Pick one candidate | Most votes wins | Largest group in each district |
| Top-two (two-round) | Pick one; a runoff follows | Top two advance, majority wins runoff | Similar to plurality, with runoff dynamics |
| IRV (single-winner ranked choice) | Rank candidates | Last place is eliminated, votes transfer, until a majority | Candidates with broad second-choice support |
| Alaska-style | Rank candidates | Plurality first round, then ranked-choice final round | Mix of the two above |
| STV (multi-winner ranked choice) | Rank candidates | Candidates reach the Droop quota; surplus and eliminated votes transfer | Groups near the quota, even if dispersed |
| Cumulative | Spread or stack m votes | Top m vote-getters win | Cohesive minorities that stack their votes |
| Limited voting | Fewer votes than seats | Top m vote-getters win | Minorities, modestly |

## 5. How a simulation runs

Most of our projects follow the same seven steps, whatever the city. The steps describe the *idea*; the lab's code runs them in five *pipeline stages* (see 5.5).

1. **Draw maps.** Generate thousands of random, legal district plans with a Markov chain (ReCom), then subsample a manageable set.
2. **Set demographics.** Pull each district’s VAP by group from Census data.
3. **Set candidates.** For each bloc and district, draw the slate size (section 3.5).
4. **Set voters.** Apply turnout rates, the cohesion table and the Dirichlet α (sections 3.2–3.4).
5. **Cast ballots.** Generate ranked, single-choice or cumulative ballots for each simulated voter.
6. **Run the election rule.** Count the ballots under each system being compared (section 4).
7. **Record winners** by bloc, repeat many times, and average.

| Step | Tool | Math or concept | Pipeline stage |
| --- | --- | --- | --- |
| 1. Draw maps | GerryChain | 5.1 | 1 (`district_generator.py`) |
| 2–4. Demographics, candidates, voters | config file + settings generator | 3.1–3.5 | 2 (`settings_generator.py`) |
| 5. Cast ballots | VoteKit ballot generators | 3.3, 3.4, 5.3 | 3 (`profile_generator.py`) |
| 6. Run the election rule | VoteKit elections | section 4, 3.6 | 4 (`simulate_elections.py`) |
| 7. Record winners | summaries and histograms | 3.7 | 5 (`summarize_results.py`) |

### 5.1 Markov chains and ReCom (how GerryChain draws maps)

A **Markov chain** is a walk where the next state depends only on the current state. **MCMC** (Markov chain Monte Carlo) uses such a walk to sample from a huge space (all valid maps) that you can't list.

**ReCom** (recombination), one step:
1. Pick two **adjacent** districts.
2. Merge them.
3. Draw a random **spanning tree** over the merged area (a tree touching every unit with no loops).
4. Cut one tree edge so both halves are within population tolerance ε.
5. Keep the new split as the new districts.

Repeat for `chain_length` steps (e.g. 1,000 to 10,000), then keep a few `num_subsamples` plans.

**The five ingredients of a GerryChain run:** `initial_state` (a Partition), `proposal` (ReCom), `constraints` (population, contiguity), `accept` function, `total_steps`.

Why bother? If the *real* map is an extreme outlier compared with thousands of neutral maps, that is evidence of gerrymandering.

### 5.2 Reproducibility and seeds

A **seed** makes “random” results repeatable. Configs use `"seed": 42`. Change the seed to check your conclusions don't depend on one lucky draw. Also, GerryChain needs `PYTHONHASHSEED=0` for fully repeatable chains. The Template's `run.py` handles this for you.

### 5.3 VoteKit in 10 lines

```python
from votekit import PreferenceProfile, Ballot
from votekit.elections import STV

profile = PreferenceProfile(ballots=[
    Ballot(ranking=[{"A"}, {"B"}, {"C"}], weight=10),
    Ballot(ranking=[{"B"}, {"A"}, {"C"}], weight=7),
    Ballot(ranking=[{"C"}, {"B"}, {"A"}], weight=5),
])
result = STV(profile, n_seats=2)
print(result)
```

Check the exact import path and signature in the docs for your installed version (the lab notes document v3.4.0).

Key VoteKit ideas:

| Concept | Meaning |
|---|---|
| `Ballot` | one ranking (or score vector) with a weight |
| `PreferenceProfile` / `RankProfile` / `ScoreProfile` | a whole electorate. Ranked rules need `RankProfile`, score rules (Approval, Cumulative, Limited) need `ScoreProfile` |
| `PreferenceInterval` | candidate weights that sum to 1 |
| Ballot generators | make synthetic profiles (Plackett–Luce, Bradley–Terry, slate models) |
| CVR loaders | read real cast-vote records |
| `FastSTV` | the faster STV used in the pipelines |
| `tiebreak` | default `None` means a tie **raises an error**. Set it (“random”, “borda”) if you expect ties |

### 5.4 Run the pipeline (your first real task)

Before you start you need two things: **GitHub access** to the `mggg` organization with an SSH key (ask your supervisor), and **uv**, the tool that installs Python and the packages for you (`brew install uv` on a Mac; see [docs.astral.sh/uv](https://docs.astral.sh/uv/) for other systems).

```bash
git clone git@github.com:mggg/Alternative-Election-Analysis-Template.git
cd Alternative-Election-Analysis-Template
uv sync
uv run run.py sample        # runs configs/sample.json
```

`sample.json` is a small example config. The first run is slow to start because the imports are heavy; that is normal. Results appear in `outputs/<run_name>/`, where `run_name` is set inside the config.

**To run a variation,** copy `configs/sample.json` to a new file, change one parameter **and the `run_name`**, then run it by its new name. Keep the `run_name` unique: stages skip themselves when their outputs already exist (see 5.5), so reusing a name would just return the old results.

### 5.5 The five stages (each reads the previous stage's output)

| # | Script | What it does |
|---|---|---|
| 1 | `district_generator.py` | GerryChain makes district plans from the geodata |
| 2 | `settings_generator.py` | Computes turnout-adjusted bloc proportions and writes VoteKit settings JSONs |
| 3 | `profile_generator.py` | Generates ballots under impulsive, deliberate, and Cambridge models |
| 4 | `simulate_elections.py` | Runs the elections (FastSTV etc.) and records winners |
| 5 | `summarize_results.py` | Builds dataframes and seat-count histograms |

Stages **skip themselves if valid outputs already exist**, so an interrupted run can restart. Delete the relevant `outputs/<run>/` subfolder to force a redo.

## 6. Validating assumptions with community partners

Every number in a model needs either a cited source or a local expert who confirmed it. You will not run these meetings in your first two weeks, so read this section for the idea and come back when you join one.

In a meeting: **state the assumption** in one plain sentence with its source, **ask one open question** (“does this match what you see locally?”), **listen for the reason** rather than just yes or no, and **write down** what you heard, who said it and which model input it changes.

Starter questions, one per model input:

| Model input             | Question to ask                                                                                                |
| ----------------------- | -------------------------------------------------------------------------------------------------------------- |
| Candidate availability  | Is there a group size below which no candidate runs, or is it gradual?                                         |
| Preferred candidate (α) | When several candidates from one community run, does one stand out or is support even?                         |
| Turnout                 | Is lower turnout uniform across the group, and does the usual explanation (such as language access) hold here? |
| Cohesion and crossover  | Do the cohesion and crossover shares match what you observe?                                                   |
| Bloc definitions        | Is this group really one political bloc?                                                                       |
| Maps and rules          | Is the concern the number of districts or where lines are drawn?                                               |

Before a meeting, learn who is in the room (mission, track record, focus) and lead with the questions closest to their expertise.

## 7. Worked example: San Diego City Council

San Diego elects 9 council members from single-member districts. Asian American and Pacific Islander (AAPI) residents are about 20% of voting-age people but hold 1 of 9 seats (11%). We ask whether other rules or maps would change that. Use this section to see sections 3–6 applied.

**2020 Census voting-age population (1,125,069 total) and current seats**

| Group | VAP | Share of VAP | Proportional seats of 9 | Seats now |
| --- | --- | --- | --- | --- |
| White | 492,808 | 43.80% | 3.94 | 4 |
| Hispanic | 298,971 | 26.57% | 2.39 | 3 |
| Asian + NHPI | 229,878 | 20.43% | 1.84 | 1 |
| Black | 79,987 | 7.11% | 0.64 | 1 |
| Others | 14,592 | 1.30% | 0.12 | 0 |
| American Indian | 8,833 | 0.79% | 0.07 | 0 |

**Model parameters we used**

- **Blocs and slates:** White, Hispanic, AAPI, Black.
- **Maps:** 100 maps subsampled from a 10,000-step chain.
- **Candidates:** slate pool drawn from a geometric variable, scaled by VAP share.
- **Turnout:** White 1.0, Black 1.0, Hispanic 0.75, AAPI 0.75. California 2024 Census data show citizen registration of about 66% (Asian) and 65% (Hispanic) vs. about 80% (White non-Hispanic).
- **Cohesion:** about 80% for each bloc’s own slate, with the crossover table from section 3.3.
- **Base system:** 3 × 3 STV.

**Scenarios tested**

- Other systems: 9 × 1 IRV, 9 × 1 Alaska, 3 × 3 cumulative, 9 × 1 top-two (close to the current system).
- Lower AAPI turnout (half or less of other blocs).
- Fewer AAPI candidates available.
- Uniform within-slate preference (α = 2) to mimic a diverse coalition.
- Regrouping American Indian and Other voters, who together are about 2% of VAP (each under 1.5%) and sit in the Black bloc.
- A single citywide district of 9 seats.

**Early findings (citywide 1 × 9 test):** STV gives AAPI at most 2 seats and about 1 on average; cumulative voting more often reaches 2 seats than limited voting or STV; STV does better with “impulsive” (less strategic) voters.

**What we asked local experts:** whether candidates really skip districts where their community is small; whether one “rockstar” candidate tends to emerge; whether AAPI turnout is uniform and still explained by language access; whether the cohesion and crossover numbers look right; and whether AAPI should be one bloc. Meeting partners included San Diegans for Justice and Common Cause California.

## 8. Your first two weeks

One day, one reading or goal, one task. Tick each box when you are done.

### Week 1: learn and run the sample

**Day 1: what the lab does**
- [ ] **Read:** sections 1 and 2.
- [ ] **Task:** look up how your own city elects its council and write it in D × m notation (for example, “7 × 1 plurality”).

**Day 2: the math and the voting systems**
- [ ] **Read:** sections 3 and 4.
- [ ] **Task:** compute the Droop quota for a 5-seat district with 10,000 valid votes. (Check: 1,667.) Then recompute the group shares and proportional seats in the section 7 table by hand or in a spreadsheet.

**Day 3: how a simulation works**
- [ ] **Read:** section 5 up to and including 5.3.
- [ ] **Task:** open a district map in Districtr (section 9) and find where one group is concentrated and where it is dispersed.

**Day 4: run it yourself**
- [ ] **Read:** sections 5.4 and 5.5.
- [ ] **Task:** install the tools, run the sample simulation and open one output folder. Write down which of the five stages made each file.

**Day 5: change one thing and report**
- [ ] **Read:** sections 6 and 7.
- [ ] **Task:** change one parameter in a copy of the sample config (for example, bloc A’s turnout from 0.7 to 0.5), run it, then write a half-page summary of what you changed, what you expected and what happened.

### Week 2: run the pipeline on your own city

You now repeat what you did in Week 1, but with data you build yourself. The lab did the same for Santa Clara, Kansas City and San Diego; the Santa Clara file `pipeline/data_generator_blocks.py` is a working example of the data steps. If the data preparation takes longer than planned, tell your supervisor, who can give you a ready-made GeoPackage so you can move on to Day 8.

**Day 6: choose the city and find its blocks**
- [ ] **Goal:** pick a small city (a few tens of thousands of residents is plenty) and note how it elects its council: seats, districts and the community group you want to follow.
- [ ] **Task:** download the 2020 TIGER/Line block shapefile for the city’s county and the city (“place”) boundary, then keep the blocks whose center falls inside the city. Open the result in a map viewer to check that it looks right.

**Day 7: add population**
- [ ] **Goal:** understand that a block shapefile has shapes only, so you must add the 2020 voting-age population (VAP).
- [ ] **Task:** join the PL 94-171 tables P1, P3 and P4 by block (`GEOID20`), either from the Census API (needs a free key) or from the San Diego repository `pipeline/data_generator_blocks.py`, which offers blocks already joined. You need total VAP and the VAP of your focal group. Save everything as a `.gpkg` with a total-VAP column and a focal-group column, like `total_vap_20` in `data/sample.geojson`.

**Day 8: configure and run**
- [ ] **Goal:** connect your data to the pipeline.
- [ ] **Task:** copy `configs/sample.json`, give it a new `run_name`, and set `geodata_path`, `population_column`, `pop_of_interest_column`, the seats and districts, candidates, turnout, cohesion and α. Start the run with `uv run run.py <your-config-name>`.

**Day 9: check the results**
- [ ] **Goal:** decide whether you trust the output.
- [ ] **Task:** compare your VAP total with the city’s official figure, and check that each cohesion row sums to 1. Then open the summaries and find how many seats your focal group wins most often, compared with its proportional seats (3.1).

**Day 10: write up and share**
- [ ] **Goal:** explain your result to someone else.
- [ ] **Task:** write a one-page summary: the city, its current system, your settings, what you found, and what you would validate locally (section 6). Share it with your supervisor.

## 9. Tools and links

GerryChain and VoteKit, with their repos and docs, are in section 1. Other tools you will use:

| Resource                               | Use                                            | Link                                                                                                                     |
| -------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Districtr                              | Draw, view and share district maps             | [districtr.org](https://districtr.org)                                                                                   |
| Alternative-Election-Analysis-Template | The pipeline you run in section 5.4            | [github.com/mggg/Alternative-Election-Analysis-Template](https://github.com/mggg/Alternative-Election-Analysis-Template) |
| TIGER/Line shapefiles                  | 2020 census block and city boundaries (Week 2) | [census.gov/cgi-bin/geo/shapefiles](https://www.census.gov/cgi-bin/geo/shapefiles/index.php)                             |
| Census API key                         | Free key for downloading PL 94-171 tables      | [api.census.gov/data/key_signup.html](https://api.census.gov/data/key_signup.html)                                       |

