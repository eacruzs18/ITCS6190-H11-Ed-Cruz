# Hands-on L11: Report

**Name:**
**Student ID:**
**Email:**

---

## Seed and commands

Seed used (your student ID):801337361

The commands you ran, in order. If you deviated from the steps in the README, say where and
why.
1. Start the cluster
```bash
docker compose up -d
```
2. Run Task 1 in original state, the run same command after task 1 is implemented.
```bash
docker cp main.py spark-master:/opt/spark/work-dir/

docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  /opt/spark/work-dir/main.py task1 /opt/spark/work-dir/shared 801337361
```
3. Run Task 2
```bash
docker cp main.py spark-master:/opt/spark/work-dir/

docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  /opt/spark/work-dir/main.py task2 /opt/spark/work-dir/shared 801337361
```
4. See the outputs
```bash
cat shared-folder/output/task1/metrics/*.csv
head -5 shared-folder/output/task2/predictions/*.csv
find shared-folder/models/full -maxdepth 2        # what a saved PipelineModel looks like
```
5. Stop the cluster
```bash
docker compose down
```

---

## Results

### Task 1: baseline and full pipeline

Paste the two lines printed by `evaluate()` (also in `shared-folder/output/task1/metrics`),
the five predictions shown and the "saved to" line.

```
baseline                         RMSE   442.8   R2  0.071
full                             RMSE   437.4   R2  0.094
+----------------------+---------------+--------+-----+----------+
|neighbourhood_cleansed|room_type      |bedrooms|price|prediction|
+----------------------+---------------+--------+-----+----------+
|Bayview               |Entire home/apt|1.0     |250.0|-23.9     |
|Bernal Heights        |Private room   |1.0     |250.0|107.4     |
|Bernal Heights        |Entire home/apt|3.0     |350.0|294.2     |
|Bernal Heights        |Private room   |1.0     |159.0|76.1      |
|Castro/Upper Market   |Private room   |1.0     |115.0|131.8     |
+----------------------+---------------+--------+-----+----------+

```

How many training and test rows did your seed give? The slides show 5,780 and 1,366 with
seed 42; why is yours different?

My seed gave `5709` training rows and `1437` test rows. `randomSplit` draws a
pseudo-random number per row from the seed. Changing the random seed alters the 
split of the dataset, resulting in a diferent count for the training and test partitions.

### Task 2: scoring with the saved model

Paste the stage list, the RMSE of the loaded model and the five predictions.

```
['StringIndexerModel', 'OneHotEncoderModel', 'VectorAssembler', 'LinearRegressionModel']
437.4 (with an R² of 0.094)
+----------------------+---------------+--------+-----+----------+
|neighbourhood_cleansed|room_type      |bedrooms|price|prediction|
+----------------------+---------------+--------+-----+----------+
|Bayview               |Entire home/apt|1.0     |250.0|-23.9     |
|Bernal Heights        |Private room   |1.0     |250.0|107.4     |
|Bernal Heights        |Entire home/apt|3.0     |350.0|294.2     |
|Bernal Heights        |Private room   |1.0     |159.0|76.1      |
|Castro/Upper Market   |Private room   |1.0     |115.0|131.8     |
+----------------------+---------------+--------+-----+----------+

```

---

## Questions

**1. Baseline versus full.** With seed 42 the lecture gets 221.6 / 0.152 for the baseline
and 220.6 / 0.160 for the full pipeline. What did you get, and how much did 97 more
features buy? The price goes from 10 to 10,000 dollars a night (the `summary()` in the
lecture): how does that affect an RMSE measured in dollars, and why do the numbers change
from seed to seed? Name one change to the pipeline you would try first (slides "Running
Example" with the model table, and "Tuning").

With seed 801337361, baseline RMSE was 442.8 and the full model was 437.4, showing that adding 
97 extra features barely improved performance. Because Airbnb prices stretch wildly from $10 to $10,000, 
extreme outliers heavily inflate the RMSE by penalizing big misses on expensive homes. 
These numbers change between seeds because each random split uses a different mix of data rows, and the
best first change to try is log-transforming the price to smooth out those extreme outliers or adding 
regularization.


**2. What is in `shared-folder/models/full`?** List its folders (`find shared-folder/models/full
-maxdepth 2`) and say what each holds. Why do we save the PipelineModel rather than the
LinearRegressionModel alone? Which stages of the saved model are fitted models and which
are plain transformers?

shared-folder/models/full contains a metadata/ folder for global pipeline info and a 
stages/ folder holding the individual steps. The entire PipelineModel is saved instead of just the 
regression model because it bundles the preprocessing steps, ensuring new data is transformed the exact same way during predictions. Among its stages, 
the string indexer, one-hot encoder, and linear regression are fitted models that learned from the data, 
while the vector assembler is a plain transformer.

**3. Only transformers at scoring time.** Task 2 ran no `fit()`. Compare the number of
predictions written with the number of test rows: if they differ, where did the rows go,
and which parameter of which stage decides it? What would you have to change in task 2 to
score a stream of listings instead of a Parquet file (slide "Scoring a Stream")?

Task 2 produced 1,436 predictions from 1,437 test rows because the missing row was dropped by 
the StringIndexer using the handleInvalid = "skip" parameter when it encountered an unknown category. 
To score a live stream instead of a Parquet file, we need to switch from batch reading (spark.read) 
to streaming (spark.readStream) and use .writeStream for continuous processing.


---

## Problems and fixes

Anything that went wrong and what resolved it. Paste the actual error message. If nothing
went wrong, say so.

When running the tasks for the first time, I got error `ModuleNotFoundError: No module named 'numpy'`. 
The problem with NumPy was that it wasn't installed inside the spark-master container, causing PySpark's 
ML library to throw a ModuleNotFoundError. When trying to install it, pip failed with a permission 
denied error because it tried to use a non-existent user home directory `(/nonexistent)`.
It was fixed by telling pip to bypass the user cache directory and install it cleanly using:
`docker exec -it spark-master pip install --no-cache-dir numpy
`
