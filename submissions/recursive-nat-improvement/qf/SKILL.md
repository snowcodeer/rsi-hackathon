---
name: quant-task-protocol
description: Solve quantitative-finance file tasks by tracing numerical conventions to the specification, inspecting inputs, building a reproducible solver, and checking both output structure and numerical meaning.
---

# Quant task protocol

The deliverables are files with specified schemas and numerical meanings.
Well-formed output is necessary but does not establish the calculation is
right. Resolve definitions before coding; verify calculations after writing.

## Phase 0: Spec to checklist (before any code)

1. Read the whole instruction once. Then write `/app/CHECKLIST.md` containing:
   - the output directory (respect `OUTPUT_DIR` if the spec mentions it;
     otherwise the path the spec gives)
   - every output file name, and for each: every column or JSON key, in order
   - every stated convention: ddof, annualisation factor, risk-free rate,
     sort order, date handling, rounding, sign conventions, units (bps vs
     decimal), day-count
   - every parameter the spec says to take from a params file rather than assume
2. Re-read the spec against the checklist. Resolve consequential ambiguities
   using the convention procedure below before choosing a default. Record
   each decision beside the output it affects.

## Phase 1: Inspect the data

Run `python3 -c` snippets or a short script to print, for every input file:
shape, columns, dtypes, first 3 rows, date range, NaN count per column,
duplicates on the key columns. Read any `params.json` or glossary fully and
print it. Do not start modelling until you know the data's quirks. Common traps:

- dates as strings, mixed formats, unsorted, duplicated
- prices vs returns, percent vs decimal, bps vs decimal
- tab-separated files with `.tsv`, quoted fields, thousands separators
- a column the spec calls X is named x_1 in the file: use the file's names
  for reading and the spec's names for writing
- joining or multiplying arrays with different date/asset order: align by
  explicit keys and verify that the join has the intended rows, not just a
  plausible shape

## Phase 2: Write one script

- Write `/app/solve.py` as a file. Do not pipe long code through `python -c`.
- Build it in sections with a `main()` that runs them in order and writes each
  output as soon as it is ready, so partial progress survives a crash.
- Load parameters from the params file when one exists. Never hard-code a
  value the spec says comes from a file.
- Sort deterministically before any operation whose result depends on order
  (rank, cumulative, rebalance dates, top-N). Use the stated tie-break; when
  absent, choose a stable tie-break and record it.
- Pass consequential statistical options explicitly. A library default is
  not evidence that the option matches the task's definition.
- Prefer exact closed forms over simulation when the spec gives a formula.
  When simulation is required, seed exactly as the spec says.
- Keep the script under a few hundred lines per file write. If it grows, split
  into helper modules and import them.

## Phase 3: Verify against the checklist (mandatory)

Run the solver, then check the outputs yourself with a short Python snippet:
list the output directory, confirm every required file is present and
non-empty, load each JSON with `json.load` and each CSV with `pandas.read_csv`,
and print the keys or columns and the row counts. Then complete the following
checks:

1. Every file in the checklist exists in the output directory. No extras needed.
2. Each CSV: columns match the required names and order. Print the
   head and the row count and compare with what the spec implies
   (one row per X means the row count equals the number of X).
3. Each JSON: keys, nesting, and value types follow the requested schema.
   Convert numpy scalars to ordinary Python values. Reject non-finite numbers
   during serialization with `allow_nan=False`; represent missing values only
   in a way the spec explicitly permits.
4. Check bounds that actually apply to the chosen model and data. Do not
   force rates or returns positive, or weights to sum to one when the task
   calls for another exposure. Check probabilities, correlations, volatility,
   model identities, and sign conventions according to their definitions.
5. Any requested consistency flags must be computed from real checks, not
   assigned a reassuring constant.
6. Boundary check every count and every schedule. For any deliverable that is
   a count (rebalances, events, periods, filings) or a date schedule, print the
   full list of events with their dates and compare the first and last against
   the spec's start rule and the parameter file (start day, lookback fill,
   warm-up). Distinguish observation time, signal availability, execution,
   return accrual, and cost booking. Derive the first eligible event from
   those rules instead of assuming an indexing offset or calendar boundary.
7. Re-run the solver and compare the generated results. Fix unintended
   randomness, unstable sorting, and state left over from the prior run.

Fix and repeat until every line of the checklist is ticked, then do Phase 3b.
Only then stop.

## Phase 3b: Independent recompute (structural OK is not correctness)

The structural checks above only prove the files are well formed. Most failed tasks
have well-formed files with wrong numbers. Before finishing, recompute at least
two key deliverables by a second, independent route and compare:

- a portfolio or strategy return: recompute from the position weights and the
  raw returns in a three-line loop, not from your pipeline's intermediate arrays
- an attribution or decomposition: verify the stated reconciliation identity,
  then independently check a component; a correct total alone cannot prove
  that the individual components use the requested definition
- a price or rate from a model: check a limiting case (zero vol, zero rate,
  deep in or out of the money, flat curve) against the closed form
- a fitted model with labelled states or regimes: label by the economic meaning
  the spec gives (for example the high-mean or low-variance state), never by
  the index the fitting library happened to assign, and print the per-state
  statistics to prove the labelling
- a count or schedule: print the first and last three events with dates

Compare routes using the same definitions, scope, and units. Investigate a
disagreement beyond the allowed tolerance; do not average the answers or
choose the more convenient one. Numerical agreement cannot resolve a shared
misreading of the specification, so recheck the convention source too.

## Resolving conventions: the spec decides, then record the choice

Most lost tasks are numerical interpretation, not malformed files. Every
consequential choice must be resolved in this order, and written into the
checklist with the reason:

1. The task's own formula, definition, or example. If the spec shows a
   formula, implement it symbol for symbol. If it names a method or variant,
   use exactly that one, even if another is more common.
2. The task's parameter file and data. Frequencies, windows, seeds, costs,
   day counts, start dates, and thresholds come from there, never from habit.
   Print each parameter you use next to the value you read.
3. Use the task's wording and observable examples to resolve what remains.
   Separate calendar frequency from execution date, gross returns from net,
   and a named estimator from a library's similarly named default. Do not
   import a convention just because it was useful on a different task.
4. If a detail truly remains unspecified, choose a defensible convention
   appropriate to this task and label it as an assumption. Keep it consistent
   throughout the solver; do not claim it is a known reference convention.

Whatever level resolved the choice, add one line to the checklist:
"<quantity>: <choice> because <spec text | parameter | literal reading |
default>". If two readings survive step 3, implement the one closest to the
spec's wording, note the alternative, and check whether any figure the spec
states (a count, a bound, an example value) discriminates between them.

Fixed mechanics that are not conventions:

- If the spec mentions `OUTPUT_DIR`, read it with `os.environ.get`, create
  the directory, and never hard-code a different path.
- Serialize numeric values using the required types: preserve integers as
  integers and convert floating scalars without prematurely rounding them.
  Keep CSV columns in the requested order.
- For random draws, match the specified seed, generator, and draw order.
  Equal seeds with different generators need not produce the same sample.
  If no generator is specified, record a reproducible choice rather than
  switching generators to obtain a preferred result.
- Retain precision throughout the calculation; round only where requested.

## Do not

- Put deliverables at the requested paths; keep solver and scratch files
  separate from the required output set.
- Do not leave a placeholder or partial file. An empty file is a failed test.
- Implement the requested method, not a more convenient substitute. A prose
  explanation does not replace an unfinished deliverable.
- Do not spend turns explaining. Spend them checking.
