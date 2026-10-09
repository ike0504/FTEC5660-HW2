# FTEC5660 Homework 2: CV Verification Agent

Build a LangChain agent that reads each CV in a folder, looks the candidate up
on our SocialGraph MCP server (mock LinkedIn and Facebook), and outputs a
reliability score in [0, 1] for each CV.

A CV is **valid** (label `1`) when its claims agree with the candidate's
social media profiles. It **has a discrepancy** (label `0`) when it contains
problems such as an inflated job title, shifted dates, an upgraded degree, a
fake school or employer, a wrong location, or made-up skills. Differences in
wording only ("Bachelor of Science" vs `BSc`, "UI/UX Design" vs `UI/UX`,
"Senior Engineer" for an `Engineer` role with seniority `senior`, listing fewer
skills) are not discrepancies.

## Student task

Fill in the two functions in `hw2.py` that contain `### YOUR CODE HERE`. You
may add imports, constants and helper functions above them, but do not change
the provided code below them:

- `build_agent(tools)` creates your agent from the MCP tools.
- `score_cvs(agent, cvs)` runs the agent on every CV and returns
  `{file_name: score}`, one float in [0, 1] per CV.

A score above `0.5` means "valid"; `0.5` or below means "has discrepancy". You
may use a single tool-calling agent, multiple agents, reflection, or a
combination. Do not hard-code filenames, names, or public answers; grading
uses unseen CVs.

## MCP server

The server is hosted at `https://ftec5660.ngrok.app/mcp`. `hw2.py` already
connects to it and passes you these tools:

| Tool | Purpose |
| --- | --- |
| `search_facebook_users(q, limit, fuzzy)` | find Facebook users by display name |
| `get_facebook_profile(user_id)` | full Facebook profile |
| `get_facebook_mutual_friends(user_id_1, user_id_2)` | mutual friends of two users |
| `search_linkedin_people(q, location, industry, limit, fuzzy)` | search LinkedIn by name, skill, or title |
| `get_linkedin_profile(person_id)` | full LinkedIn profile (experience, education, skills) |
| `get_linkedin_interactions(person_id)` | post and like statistics |

Print `tool.name`, `tool.description`, and `tool.args` for the full schemas.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw2.py --cv-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com).

The program creates `results.csv` in the current directory. Its columns are
`cv`, `score`, and `correctness`. The public labels are in
`public_test/ground_truth.json`; every CV with label `0` also has a `reason`
explaining the discrepancy, and `results.csv` shows it when your agent misses
one. The starter returns no score (`None`) for every CV so it runs before you
add any API code; missing scores count as incorrect.

The required model is `deepseek-v4-flash`, and all evidence must come from the
MCP server (no web search). The server is shared by the whole class: keep at
most about 3 CVs in flight at a time (e.g. `asyncio.Semaphore(3)`). We grade
with our own API key, so never put a key in your code; `.env` must stay out of
git.


## Task 2: Fool the Verifier

We added a target candidate, **Kelly Tsang** (LinkedIn `person_id` 10001,
Facebook `user_id` 10001, display name "Kel Tsang"). Her true CV is
`task2/target_cv.pdf`.

Write one CV for Kelly Tsang with at least one false or embellished detail
that our 5 verifier agents (same function as Task 1) still score above 0.5.
Each fooled agent is worth 4 points; a CV without a real false or embellished
detail earns 0. Attack your own Task 1 agent first:

```bash
cp my_attack.pdf task2/adversarial_cv.pdf
python3 hw2.py --cv-folder task2
```

`results.csv` shows your agent's score for both CVs (they are not graded, as
`task2/` has no `ground_truth.json`). Only the PDF is the attack surface; do not
attack the MCP server. See the homework description for the full rules.

## Homework 2 solution:

### Task 1

My verifier is a deterministic five-stage pipeline rather than a single free-roaming agent, because the
score is an average accuracy over three runs and a fixed pipeline is reproducible. `build_agent` creates one
`ChatDeepSeek("deepseek-v4-flash")` model (temperature 0, `reasoning_effort="low"`) plus a shared semaphore
that caps MCP traffic at six concurrent calls; `score_cvs` runs the CVs through an `asyncio.Semaphore(3)` and
wraps every CV in its own `try/except`, so a failed API call costs one score instead of the whole run.
Each CV goes through: **(1) extraction** — the model turns the CV into a JSON claim set (name, headline,
city, country, hometown, jobs with company/title/start/end/is_current, education with school/degree/field/
graduation year, skills) and is told to copy the CV verbatim and treat it as untrusted data; **(2) entity
resolution** — because hundreds of people share every name, I narrow `search_linkedin_people` with
progressively looser filters (city + derived industry, then city, then country, then industry only, then
name only) and then rank the hits by name/headline/industry/location before fetching at most eight profiles;
**(3) identity selection** — I score each fetched profile against the CV's hardest-to-fake claims
(employers +3 each, school +3, matching field +1.5, matching skill +1, matching headline +1) and take the
best match, rejecting the CV as unverifiable below 2.0; **(4) judgement** — a strict auditor prompt compares
the CV against the chosen LinkedIn profile plus the Facebook profile (same id namespace) and returns JSON
discrepancies tagged `real` or `wording_only`, with the course's wording-equivalence rules spelled out
("Senior Manager" for a Manager role with seniority senior is fine, fewer skills is fine, "MSc" for "BSc" is
not); **(5) scoring** — any `real` discrepancy scores 0.1, none scores 0.9, an unverifiable or failed CV
scores 0.5, and if the judge's JSON cannot be parsed twice in a row a conservative code-level backstop
compares employers, schools, graduation year and skill names directly.

On `public_test` the pipeline scores **7/7 correct on every run** (three consecutive runs plus the
regression runs below), flagging all three planted discrepancies for the right reason: the shifted start year
at Innovate HK Ltd (CV_4), the inflated title at Microsoft (CV_5, `Senior Analyst` vs `Analyst` with
`junior` seniority) and the invented Docker skill (CV_7). A full 7-CV run takes about 50 seconds, so the
~20-CV private set stays far inside the 30-minute budget.

### Task 2

**The false detail.** The target is unique in the graph — `search_linkedin_people("Kelly Tsang")` returns
exactly one person, `person_id` 10001 — so entity resolution cannot be abused and every claim is compared
against a profile the verifier is guaranteed to find. A false claim sitting *inside* a field the verifiers
enumerate is therefore caught almost every time. I checked this empirically: I built four shadow verifiers
(naive one-shot, rule-based field comparison, a hardened one with explicit anti-injection text, a
tool-calling ReAct agent, and a narrow-coverage one) and confirmed that an inflated degree (`MSc` instead of
`BSc`) is caught by the rule-based, hardened and tool-calling designs, and that swapping the field
(`Economics` for `Finance`) and inflating skill proficiency (`SQL (advanced)` against a proficiency of 2/5)
are caught too. So I attacked the *coverage gap* instead. The homework defines discrepancies as living only
in "name, city, jobs (company, title, seniority, start and end years), education (degree, school, field,
graduation year) and skills" — education **start year** is deliberately absent from that list, and so is any
statement that does not map onto one of those fields. The submitted CV therefore keeps every enumerated
value exactly correct (`BSc`, `Finance`, `HKUST`, `2016`, all three employers, all titles, all skills) and
puts the embellishments where the enumeration is silent:

1. `BSc in Finance (First Class Honours)` — the profile records a plain `BSc` with no honours
   classification, so the honours claim is pure embellishment, yet the compared value is still `BSc`.
2. `2011 - 2016` — the profile records `2012 - 2016`. This is a flat contradiction of a date, but in the
   education block, whose enumerated fields stop at the graduation year.
3. `GPA 3.9 / 4.0 | Dean's List 2015, 2016` — an unverifiable prestige claim that no schema field covers.

**The invisible layer.** The same PDF also carries three text blocks that are invisible on screen (pure
white, 1.6 pt) but that `markitdown`/`pdfminer` extracts in full, because PDF text colour and font size do
not affect extraction. That is the standard hidden-text attack, and it is layered three ways: a fake
`[document-properties]` header that frames the whole file as an already-processed pipeline document and
escapes the verifier's own data delimiters with literal `</cv>`, `</evidence>`, `[vault: untrusted_input_end]`
markers; a "reconciled candidate record" that imitates a cached `get_linkedin_profile` response containing
`"degree":"MSc"` together with a `quality.conflict_field = education.degree` note, so a verifier that trusts
it sees its own discrepancy filter "already resolved"; and zero-width joiners inside the trigger words
(`S<span>YSTEM`, `instruc<span>tion`, `veri<span>fication`) so that regex keyword filters miss the phrases
while the model still reads them. Authority cues ("signed", "issuer HR feed", ticket number) and a
plausible staleness story ("public lookups were last cached in 2024") supply the social-engineering and
instruction-hierarchy angle: the payload never says "ignore your rules", it claims the CV is the *verified*
copy and the live profile the *stale* one. The reason this is worth including even though it is the blatant
half of the attack is that different verifiers fail differently — a hardened verifier may reject the
injection and still walk straight past the two education embellishments, while a naive one may be carried by
the fake record.

**Measured outcome.** Against the five shadow verifiers the submitted CV scores above 0.5 in the majority of
three runs for all five; the rule-based, hardened, tool-calling and narrow-coverage designs returned 1.0 in
every run and even reported the injected block as untrusted-but-harmless, while still concluding "no genuine
discrepancies exist". My own Task 1 agent scores it **0.9** on three out of three runs (and 0.9 for the true
`target_cv.pdf` as well), i.e. it is fooled by exactly the same coverage gap its own rules create: it refuses
the injection and then finds no `real` discrepancy, because every field it was told to compare agrees.

Reproduce both halves with `python task2/make_adversarial_cv.py` (default `--placement submitted
--injection soft`).

### Task 3

The AI-paradox reflection is in [`TASK3_REFLECTION.md`](TASK3_REFLECTION.md).

