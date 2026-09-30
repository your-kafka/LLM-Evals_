# LLM Text-to-SQL Evals

An execution-accuracy benchmark that tests how well large language models turn plain-English questions into correct SQL. It runs on real data: every ball of the IPL (Indian Premier League cricket) from 2021 to 2024.

Five models answer the same 20 hard analytical questions. Each generated query runs against a SQLite database, and its result set is compared with the result of a gold reference query. A model gets credit for returning the right data, not for writing SQL that looks like the gold query.

## Results

| Model | Score | Hard (8) | Brutal (12) |
|---|---|---|---|
| Grok 4.5 | **18/20 (90%)** | 8/8 | 10/12 |
| Claude Sonnet 5 | 17/20 (85%) | 8/8 | 9/12 |
| GPT-5.6 Terra | 16/20 (80%) | 7/8 | 9/12 |
| MiniMax M3 | 13/20 (65%) | 7/8 | 6/12 |
| Kimi K3 | 11/20 (55%) | 6/8 | 5/12 |

Per-question verdicts and the SQL each model wrote are in [`eval_results.csv`](eval_results.csv).

**Reading the numbers**

- **No model solved #10 (economy rate) or #11 (bowling average), but the misses are close.** Every model that returned SQL for them named the correct bowler. The values differed because the models defined "runs conceded" differently from the gold query, which counts every run (`SUM(total_runs)`). Most models left out byes and leg-byes, which cricket scoring doesn't charge to the bowler. These two questions test an ambiguous definition as much as SQL skill.
- **Kimi K3 and MiniMax M3 lost points to empty responses.** In 9 of their 10 `sql_error`s the model returned no SQL at all, rather than broken SQL. The `max_tokens=800` cap in `main.py` may be the cause. Treat those two scores as a lower bound.

## How it works

1. **Build the database.** `db.py` filters the Kaggle IPL CSVs down to 2021–2024 and writes `ipl_2021_2024.db`. It has two linked tables: `matches` (279 rows) and `deliveries` (67,303 rows, one per ball).
2. **Extract the schema.** `schema_extractor.py` writes the exact `CREATE TABLE` statements to `schema.sql`. That text goes into the prompt.
3. **Build the golden dataset.** Each question is paired with a gold SQL query. The query's executed result is stored as JSON, so scoring compares data, not SQL text.
4. **Generate.** Each model receives the schema and one question through [OpenRouter](https://openrouter.ai) (temperature 0) and is asked for a single SQLite query. Markdown fences and extra prose are stripped from the reply.
5. **Execute and evaluate.** The generated SQL runs against the database, and `evaluator.py` compares its result with the gold result.

## Evaluation method

`evaluator.py` measures **execution accuracy** by comparing values only:

- **Column names are ignored.** Only the values in the result set count.
- **Numbers are compared numerically.** `24395` equals `24395.0`, and floats are rounded to 4 decimals to absorb noise.
- **Row order is ignored** unless a question is marked `order_sensitive` (only #12 is).
- **One column more or fewer than the gold is accepted.** For "who scored the most?", both `name` and `name, runs` count as correct.

Every row in `eval_results.csv` gets one of these outcomes: `match`, `mismatch`, `sql_error` (the query failed or was empty), `gen_error` (the API call failed), or `compare_error`.

## The golden dataset

There are 20 questions in two tiers, graded by how many traps each question stacks:

| Tier | Questions | Traps |
|---|---|---|
| **hard** | #1–8 | 1–2: a ratio, a NULL-aware filter, a threshold, *or* a self-join |
| **brutal** | #9–20 | 3+: ratios combined with NULL handling, `HAVING` thresholds, subqueries, and cricket rules |

These traps separate the models:

- **NULL-sensitive filtering.** `extras_type NOT IN (...)` silently drops every row where `extras_type` is NULL.
- **Legal-ball counting.** Wides and no-balls must be excluded from balls faced and balls bowled.
- **Wicket attribution.** Run outs, retired hurt, retired out, and obstructing the field don't count as the bowler's wickets.
- **Rates and ratios.** Strike rate, economy, bowling average, and percentages.
- **Innings and phase logic.** The powerplay is overs 0–5, the death overs are 15–19, and batting first differs from chasing.

## Project structure

| File | Purpose |
|---|---|
| `main.py` | Runs the full evaluation across all models and writes `eval_results.csv` |
| `evaluator.py` | Result-set comparison logic |
| `model_openrouter_slug.py` | The models under test, as OpenRouter slugs |
| `first_test.py` | Smoke test that sends one question to one model and prints the raw SQL |
| `db.py` | Builds `ipl_2021_2024.db` from the raw CSVs in `data/` |
| `schema_extractor.py` | Prints the schema and sample rows, and writes `schema.sql` |
| `golden_dataset_generator.py` | The 20 questions and their gold SQL. Run it directly to validate them against the database |
| `make_golden_dataset.py` | Runs the gold SQL and writes the questions and gold results to CSV |
| `golden_dataset.csv` | The golden dataset that `main.py` reads |
| `eval_results.csv` | Verdicts and generated SQL for every model and question |
| `schema.sql` | The schema text used in the prompt |
| `ipl_2021_2024.db` | The prebuilt SQLite database |
| `data/` | Raw IPL `matches.csv` and `deliveries.csv` |

## Getting started

**Prerequisites:** Python 3.10+ and an [OpenRouter API key](https://openrouter.ai/keys).

```bash
git clone https://github.com/your-kafka/LLM-Evals-2.git
cd LLM-Evals-2

python3 -m venv .venv && source .venv/bin/activate
pip install langchain-openrouter python-dotenv pandas

echo "OPENROUTER_API_KEY=sk-or-..." > .env
```

The database, schema, and golden dataset are already committed, so you can run the evaluation straight away:

```bash
python first_test.py   # optional: check that a single model call works
python main.py         # full run over every model and question, writes eval_results.csv
```

To test other models, edit the `MODELS` list in `model_openrouter_slug.py`. Any [OpenRouter model slug](https://openrouter.ai/models) works.

### Rebuilding the data (optional)

```bash
python db.py                         # rebuild ipl_2021_2024.db from data/*.csv
python schema_extractor.py           # re-export schema.sql
python golden_dataset_generator.py   # check that all 20 gold queries run
python make_golden_dataset.py        # writes golden_hard.csv
mv golden_hard.csv golden_dataset.csv
```

`make_golden_dataset.py` writes `golden_hard.csv`, but `main.py` reads `golden_dataset.csv`, which is why the last step renames the file.

`schema_extractor.py` also has a `RENAME` switch. It builds a copy of the database with renamed tables and columns, so models can't lean on memorized Kaggle column names. This guards against data contamination.

## Author

**Ramkrishna Rathore**
[LinkedIn](https://www.linkedin.com/in/ramkrishna-rathore-1a843b22a)

## Acknowledgements

The IPL data comes from the Kaggle IPL complete dataset.
