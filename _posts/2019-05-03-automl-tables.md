---
layout: post
title: "AutoML Tables and the 'Chicago Taxi Trips' dataset"
categories:
- AutoML
- ML
tags: automl ml
date: 2019-05-03
---


[AutoML Tables](https://cloud.google.com/automl-tables/) was recently [announced](https://cloud.google.com/blog/products/ai-machine-learning/expanding-google-cloud-ai-to-make-it-easier-for-developers-to-build-and-deploy-ai) as a new member of GCP's [family](https://cloud.google.com/automl/) of AutoML products. It lets you automatically build and deploy state-of-the-art machine learning models on structured data.

I thought it would be fun to try AutoML Tables on a dataset that's been used for a number of recent
[TensorFlow](https://www.tensorflow.org)-based examples: the
['Chicago Taxi Trips' dataset](https://pantheon.corp.google.com/marketplace/details/city-of-chicago-public-data/chicago-taxi-trips), which is one of a large number of
[public datasets hosted with BigQuery](https://cloud.google.com/bigquery/public-data/).

These examples use this dataset to build a neural net model (specifically, a ["wide and deep" TensorFlow](https://www.tensorflow.org/api_docs/python/tf/estimator/DNNLinearCombinedClassifier) model) that predicts whether a given trip will result in a tip > 20%.  (Many of these examples also show how to use the [TensorFlow Extended (TFX)](https://www.tensorflow.org/tfx) libraries for things like data validation, data preprocessing, and model analysis).

<figure>
<a href="/images/pipelines_acc.png" target="_blank"><img src="/images/pipelines_acc.png" /></a>
<figcaption><br/><i>Metrics from running the <a href="https://kubeflow.org">Kubeflow Pipelines</a> <a href="https://github.com/kubeflow/pipelines/tree/master/samples/tfx">version</a> of the TFX-based 'Chicago Taxi Data' <a href="https://github.com/tensorflow/tfx/blob/master/tfx/examples/chicago_taxi/README.md">example</a>.</i></figcaption>
</figure>

<p></p>

We can't directly compare the results of the following AutoML experimentation to those other examples, since they're using a different dataset and model architecture. However, we'll use a roughly similar set of input features and stick to the spirit of these other examples by doing binary classification on the tip percentage, and to that end, we'll generate a version of the taxi trips dataset that has a new column reflecting whether or not the tip was > 20%.  We'll also do a bit of 'bucketing' of the lat/long information using the new BigQuery GIS functions, and weed out rows where either the fare or trip miles are not > 0.

### Create a BigQuery table 

AutoML Tables makes it easy to ingest data from [BigQuery](https://cloud.google.com/bigquery/).
So, the first thing we'll do is run a BigQuery query to generate this new version of the dataset. Paste the following SQL into the [BigQuery query window](https://console.cloud.google.com/bigquery), or [use this URL](https://console.cloud.google.com/bigquery?sq=467744782358:ae38c8baf54a46489536ae486979a1bc).

> Note: when I ran this query, it processed 15.66 GB of data. 


```sql
WITH
  taxitrips AS (
  SELECT
    trip_start_timestamp,
    trip_end_timestamp,
    trip_seconds,
    trip_miles,
    pickup_census_tract,
    dropoff_census_tract,
    pickup_community_area,
    dropoff_community_area,
    fare,
    tolls,
    extras,
    trip_total,
    payment_type,
    company,
    pickup_longitude,
    pickup_latitude,
    dropoff_longitude,
    dropoff_latitude,
    IF((tips/fare >= 0.2),
      1,
      0) AS tip_bin
  FROM
    `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE
    trip_miles > 0
    AND fare > 0)
SELECT
  trip_start_timestamp,
  trip_end_timestamp,
  trip_seconds,
  trip_miles,
  pickup_census_tract,
  dropoff_census_tract,
  pickup_community_area,
  dropoff_community_area,
  fare,
  tolls,
  extras,
  trip_total,
  payment_type,
  company,
  tip_bin,
  ST_AsText(ST_SnapToGrid(ST_GeogPoint(pickup_longitude,
        pickup_latitude),
      0.05)) AS pickup_grid,
  ST_AsText(ST_SnapToGrid(ST_GeogPoint(dropoff_longitude,
        dropoff_latitude),
      0.05)) AS dropoff_grid,
  ST_Distance(ST_GeogPoint(pickup_longitude,
      pickup_latitude),
    ST_GeogPoint(dropoff_longitude,
      dropoff_latitude)) AS euclidean
FROM
  taxitrips
LIMIT
  100000000
```

You can see the use of the `ST_SnapToGrid` function to "bucket" the lat/long data. (Currently, AutoML Tables doesn't support the BigQuery GIS data types, so we're converting those to text and will treat them categorically). We're also generating a new euclidean distance measure between pickup and dropoff using the `ST_Distance` function.

When the query has finished running, export the results to a table in your own project, by clicking on "Save Results".

<figure>
<a href="/images/bq_save_results.png" target="_blank"><img src="/images/bq_save_results.png" width="500"/></a>
<figcaption><br/><i>Save the query results to a table in your own project.</i></figcaption>
</figure>

<p></p>

### Import your new table as an AutoML dataset

Then, return to the [AutoML panel in the Cloud Console](https://console.cloud.google.com/automl-tables/datasets), and import your new table (the one in your own project), as a new AutoML Tables dataset.

<figure>
<a href="/images/create_dataset.png" target="_blank"><img src="/images/create_dataset.png" width="500"/></a>
<figcaption><br/><i>Import your new BigQuery table to create a new AutoML Tables dataset.</i></figcaption>
</figure>

<p></p>

### Specify a schema, and launch the AutoML model training job

After the import has completed, you'll next specify a schema for your dataset. Here is where you indicate which column is your 'target' (what you'd like to learn to predict), as well as the column types. 

We'll use the `tip_bin` column as the _target_.  Recall that this is one of the new columns we created when generating the new BigQuery table.  So, the ML task will be to learn how to predict — given other information about the trip — whether or not the tip will be over 20%.
Note that once you select this column as the target, AutoML automatically suggests that it should build a classification model, which is what we want.

Then, we'll adjust some of the column types.  AutoML does a pretty good job of inferring what they should be, based on the characteristics of the dataset.  However, we'll set the 'census tract' and 'community area' columns (for both pickup and dropoff) to be treated as *categorical*, not numerical. 

<figure>
<a href="/images/schema.png" target="_blank"><img src="/images/schema.png" width="90%"/></a>
<figcaption><br/><i>Specify the input schema for your training job.</i></figcaption>
</figure>

<p></p>

We can view an analysis of the dataset as well.  Some rows have a lot of missing fields.  We won't take any further action for this example, but if this was your own dataset, this might indicate places where your data collection process was problematic or where your daataset needed some cleanup.

<figure>
<a href="/images/analyze.png" target="_blank"><img src="/images/analyze.png" width="90%"/></a>
<figcaption><br/><i>AutoML Table's analysis of the dataset.</i></figcaption>
</figure>

<p></p>

Now we're ready to kick off the training job. We need to tell it how many node-hours to spend.  Here I'm using 3, but you might want to use just 1 hour in your experiment. [(Here's](https://cloud.google.com/automl-tables/pricing?_ga=2.214193684.-287350488.1556733758) the (beta) pricing guide.)
We'll also tell AutoML which (non-target) columns to use as input.  Here, we'll indicate to drop one column,
the `trip_total`. This value is correlated with tip, so for the purposes of this experiment, its inclusion is 'cheating'.

<figure>
<a href="/images/train_setup.png" target="_blank"><img src="/images/train_setup.png" width="500"/></a>
<figcaption><br/><i>Setting up an AutoML Tables training job.</i></figcaption>
</figure>


### Evaluating the AutoML model

When the training completes, you'll get an email notification. AutoML automatically generates and displays model evaluation metrics for you. We can see, for example, that the model accuracy is 90.6% and the AUC ROC is 0.954. 

<figure>
<a href="/images/automl_eval1.png" target="_blank"><img src="/images/automl_eval1.png" width="95%"/></a>
<figcaption><br/><i>AutoML Tables model evaluation info.</i></figcaption>
</figure>

<p></p>

It also generates a *confusion matrix*... 

<figure>
<a href="/images/automl_confusion_matrix.png" target="_blank"><img src="/images/automl_confusion_matrix.png" width="90%"/></a>
<!-- <figcaption><br/><i>xxx.</i></figcaption> -->
</figure>

<p></p>

...and a histogram of the input features that were of most importance. This histogram is kind of interesting: it suggests that `payment_type` was most important.  (My guess: people are more likely to tip if they're putting the fare on a credit card, where a tip is automatically suggested).  It looks like the pickup and dropoff info was not that informative, though information about trip distance was a bit more so.

<figure>
<a href="/images/automl_feature_impt.png" target="_blank"><img src="/images/automl_feature_impt.png" width="90%"/></a>
<figcaption><br/><i>The most important features in the dataset, for predicting the given target.</i></figcaption>
</figure>

### Use your new model for prediction

[** ... **]

<figure>
<a href="/images/evaluated_examples.png" target="_blank"><img src="/images/evaluated_examples.png" width="99%"/></a>
<figcaption><br/><i>xxxx</i></figcaption>
</figure>

<p></p>

### Summary

[** ... **]
