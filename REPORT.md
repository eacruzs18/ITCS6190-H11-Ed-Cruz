# Hands-on L11: Report

**Name:**
**Student ID:**
**Email:**

---

## Seed and commands

Seed used (your student ID):

The commands you ran, in order. If you deviated from the steps in the README, say where and
why.

```bash
docker compose up -d
docker cp main.py spark-master:/opt/spark/work-dir/
docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  /opt/spark/work-dir/main.py task1 /opt/spark/work-dir/shared <SEED>
docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  /opt/spark/work-dir/main.py task2 /opt/spark/work-dir/shared <SEED>
cat shared-folder/output/task1/metrics/*.csv
head -5 shared-folder/output/task2/predictions/*.csv
find shared-folder/models/full -maxdepth 2
docker compose down
```

No deviations from the README steps.

---

## Results

### Task 1: baseline and full pipeline

Paste the two lines printed by `evaluate()` (also in `shared-folder/output/task1/metrics`),
the five predictions shown and the "saved to" line.

```
<PASTE task1 OUTPUT: seed line, baseline/full evaluate() lines, 5 predictions, saved-to line>
```

How many training and test rows did your seed give? The slides show 5,780 and 1,366 with
seed 42; why is yours different?

My seed gave `<TRAIN>` training rows and `<TEST>` test rows. `randomSplit` draws a
pseudo-random number per row from the seed, so a different seed puts different rows (and a
slightly different count) in each set; the 80/20 ratio is only approximate.

### Task 2: scoring with the saved model

Paste the stage list, the RMSE of the loaded model and the five predictions.

```
<PASTE task2 OUTPUT: stages: ['StringIndexerModel', 'OneHotEncoderModel', 'VectorAssembler', 'LinearRegressionModel'], RMSE line, row counts, 5 predictions>
```

The RMSE of the loaded model is identical to the "full" RMSE of task 1, since it is the same
fitted PipelineModel applied to the same test rows.

---

## Questions

**1. Baseline versus full.** With seed 42 the lecture gets 221.6 / 0.152 for the baseline
and 220.6 / 0.160 for the full pipeline. What did you get, and how much did 97 more
features buy? The price goes from 10 to 10,000 dollars a night (the `summary()` in the
lecture): how does that affect an RMSE measured in dollars, and why do the numbers change
from seed to seed? Name one change to the pipeline you would try first (slides "Running
Example" with the model table, and "Tuning").

I got baseline `<RMSE_B>` / `<R2_B>` and full `<RMSE_F>` / `<R2_F>`. The ~97 extra
features (one-hot categories plus numeric columns) buy only a small improvement, a few
dollars of RMSE and a few points of R2. RMSE squares the errors, so the few listings priced
in the thousands dominate it: a single 10,000-dollar listing mispredicted by several
thousand dollars weighs more than hundreds of ordinary listings. Which of those outliers
fall in the test set depends on the split, which is why the numbers move from seed to seed.
The first change I would try is predicting `log(price)` and exponentiating the prediction
(the model table in "Running Example"), which tames the outliers; next, a tree-based model
(RandomForestRegressor) tuned with CrossValidator and a ParamGrid ("Tuning").


**2. What is in `shared-folder/models/full`?** List its folders (`find shared-folder/models/full
-maxdepth 2`) and say what each holds. Why do we save the PipelineModel rather than the
LinearRegressionModel alone? Which stages of the saved model are fitted models and which
are plain transformers?

`find shared-folder/models/full -maxdepth 2` shows:

- **`metadata/`**: JSON describing the PipelineModel (class, Spark version, uid, ordered
  list of stage uids).
- **`stages/`**: one sub-folder per stage, prefixed by its index:
  `0_StringIndexer_...` (labels learned for each string column), `1_OneHotEncoder_...`
  (category sizes), `2_VectorAssembler_...` (input/output column params only),
  `3_LinearRegression_...` (coefficients and intercept in `data/`, params in `metadata/`).

We save the PipelineModel because the LinearRegressionModel alone expects a ready-made
`features` vector; the scoring job would have to rebuild the indexing, encoding and
assembly with exactly the same labels and column order, which is error-prone. The
PipelineModel carries the whole preprocessing with it. Fitted models: `StringIndexerModel`,
`OneHotEncoderModel`, `LinearRegressionModel`. Plain transformer: `VectorAssembler`.


**3. Only transformers at scoring time.** Task 2 ran no `fit()`. Compare the number of
predictions written with the number of test rows: if they differ, where did the rows go,
and which parameter of which stage decides it? What would you have to change in task 2 to
score a stream of listings instead of a Parquet file (slide "Scoring a Stream")?

Task 2 wrote `<N_PRED>` predictions for `<TEST>` test rows. Any missing rows are test
listings whose value in a string column (e.g. a rare `property_type` or
`neighbourhood_cleansed`) never appeared in the training set: the `StringIndexer` stage was
fitted with `handleInvalid="skip"`, so it drops those rows instead of failing
(`"keep"` would put them in an extra bucket, `"error"` would abort). To score a stream I
would only change the input and output: read with `spark.readStream.schema(schema).parquet(...)`
(or Kafka) instead of `spark.read`, keep `model.transform(...)` unchanged, and write with
`writeStream` (format, checkpoint location, `start()`/`awaitTermination()`) instead of
`write`. The RMSE cannot be computed on an unbounded stream with the batch evaluator.


---

## Problems and fixes

Anything that went wrong and what resolved it. Paste the actual error message. If nothing
went wrong, say so.

<FILL IN after running; e.g. "Nothing went wrong.">
