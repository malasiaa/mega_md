# mega_md

Context: I'm working on a binary LightGBM classifier that outputs a probability (will the
recipient be at home?). The data is ~500M time-ordered rows, and recipients/addresses repeat
over time. I want help with data preparation and feature engineering. Treat the points below as
guidance and good practice, and use your judgment when the codebase or data suggests otherwise.

[Add: how to run the pipeline, where the data and feature code live, the prediction moment,
how the label is defined, and the current baseline metrics.]

Start by understanding before changing anything
- Map the flow from raw sources to the training table: data sources, joins, filters, feature
  code, label creation, and what happens at serving time. Produce a feature inventory (name,
  source, how it is computed, which information it relies on, whether it is available at
  prediction time).
- Clarify the framing: when the model is called, what one row represents (parcel, attempt or
  stop), and exactly what counts as "not at home" versus other failure reasons. Many
  data-quality and leakage questions are answered here.
- Profile the data before deciding anything: row counts and label rate per day, missingness
  by feature and over time, duplicates, join fan-out, sentinel values, outliers, category
  levels and how they change, and gaps from outages or backfills.

Data preparation
- Keep the training table faithful to what production will see. Features should reflect only
  the information available at the prediction moment, including ingestion delays. A useful
  sanity test is to remove all events after a row's prediction time and check that its
  features don't change.
- Work with time in mind: store timestamps in UTC, derive local hour, weekday and holidays
  with an explicit time zone, and check daylight-saving change days. Calendars, holidays and
  weather or traffic forecasts known at prediction time are good inputs.
- Clean with intent: turn sentinel values into missing, deduplicate at the unit of decision,
  validate joins with row-count checks, and fix dtypes. LightGBM handles missing values
  natively, so imputing is usually unnecessary. Keep a missingness indicator only when the
  absence itself is informative.
- Handle categoricals through pandas categoricals with a fixed, versioned mapping shared by
  training and serving. Decide how rare and unseen levels are treated.
- For the first weeks of data, history features are short or empty. Decide whether to trim
  them or flag them so the share of "new recipient" rows resembles production.
- Use one feature code path for training and serving, and document the expected schema
  (names, dtypes, categories, ranges) so mismatches are caught early.
- When sampling for faster experiments, compute features on the full data first, then sample
  by recipient or by day so histories stay intact.
- Use data checks (schema, ranges, null rates, daily counts) to catch problems early.

Feature engineering
- Brainstorm from how the domain works. For this problem, think about:
  - recipient history: past at-home rate (smoothed), recency, number of attempts, streaks;
  - time patterns: hour, weekday, holidays, time since the last attempt, days to the weekend;
  - address and place: building type, business vs residential, zone-level behaviour;
  - delivery context: parcel type, service level, delivery window, route characteristics.
- Use relative and cyclic time features (hour, weekday, time since an event) rather than
  absolute dates, and prefer history features over raw IDs so the model generalises to new
  recipients.
- Build aggregates with an explicit lookback window and a smoothing prior toward the
  population or zone average, so they stay sensible for recipients with little history. Also
  provide the sample size behind each aggregate.
- Consider ratios, differences from a recipient's own typical behaviour, and counts over
  several windows (e.g. 7/30/90 days). Trees handle interactions themselves, so favour a few
  informative signals over many near-duplicates.
- Pay attention to features with a natural direction; they can be good candidates for
  monotone constraints later.
- Track cost and availability for each feature (latency, pipeline fragility, personal data)
  and prefer features that earn their keep.
- Test each new idea as a small experiment: state the hypothesis, compare against the
  baseline on time-based validation windows, and look at the effect on new recipients and
  key segments, not only the global metric.

What I'd like from you
- A short written summary of what you found (pipeline map, feature inventory, data-quality
  observations, and anything that looks risky or inconsistent), with file references.
- A prioritised list of improvements to data preparation and candidate new features, with the
  expected benefit and effort of each.
- Then, after we agree on priorities, implementation in small, reviewable steps, each
  accompanied by evidence (the command you ran and its output) that it behaves as intended.
