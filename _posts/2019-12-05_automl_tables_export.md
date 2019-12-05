---
layout: post
title: "AutoML Tables: Exporting and serving your trained model to Cloud Run"
categories:
- AutoML
- ML
tags: automl cloud_ml 
date: 2019-12-05
---


## Introduction

Google Cloud’s [AutoML Tables][1] lets you automatically build and deploy state-of-the-art machine learning models using your own structured data.
Recently, Tables launched a feature to let you [export][2] your full custom model, packaged such that you can serve it via a Docker container. This lets you serve your models anywhere that you can run a container.

In this example, we'll show how you can package an exported Tables model to serve on [Cloud Run][3]. With Cloud Run, your model serving automatically scales out with traffic, and scales to 0 when it’s not being used.  We’ll also show how you can examine your trained custom model in [TensorBoard][4].

For the following steps, the [documentation][5] provides more detail.
We'll use the [Cloud Console UI][6], but all of these steps could also be accomplished by accessing the API via the command line or using the [AutoML Tables client libraries][7].

## Create a Dataset and edit its schema

The first step in training a Tables model is to create a *dataset*, using your data.
For this example, we'll use data that is essentially a join of the [London Bikes][8] and [NOAA weather][9] public datasets in BigQuery, with some additional processing to derive additional GIS and day-of-week fields. (If you like, you can follow along with your own tabular dataset instead, but you will need to construct your own prediction instances as well).

Visit the [Tables page][10] in the Cloud Console, and enable the API as necessary.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/tables_export/enable_api.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/tables_export/enable_api.png" width="40%"/></a>
<figcaption><br/><i>Enable the AutoML Tables API.</i></figcaption>
</figure>

Then, create a new Tables *dataset*.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/tables_export/create_dataset.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/tables_export/create_dataset.png" width="50%"/></a>
<figcaption><br/><i>Create a new Tables dataset.</i></figcaption>
</figure>

Import your data into the dataset. To ingest the example data, select "Import data from BigQuery".  Then, as shown in the figure below, use `aju-dev-demos` as the BigQuery Project ID, `london_bikes_weather` as the dataset ID, and `bikes_weather` as the table name.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/tables_export/import_data.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/tables_export/import_data.png" width="60%"/></a>
<figcaption><br/><i>Import the <code>bikes_weather</code> BigQuery table into the dataset.</i></figcaption>
</figure>

### Edit the dataset’s schema

Once the import is complete, edit the dataset schema. We'll need to change a few of the inferred types. Make sure your schema reflects that in the figure below. In particular, change `bike_id`, `end_station_id`, `start_station_id`, and `loc_cross` to be of type *Categorical*. (Note that useful stats are generated for the columns).

Then, we'll set `duration` as the _target_ column.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/tables_export/schema.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/tables_export/schema.png" width="90%"/></a>
<figcaption><br/><i>Adjust the dataset schema.</i></figcaption>
</figure>


## Train and export your Tables model

Now you're ready to train a model on that dataset.  After the model is trained, you will be able not only to deploy it to the Cloud AI Platform, where you can access it via API, but also can *export* it to run in a container.

### Train the model

We'll train a model to predict ride  `duration` given all the other dataset inputs.  So, we'll be training a [regression][11] model.
For this example, enter a training budget of 1 hours, and include all the other feature columns.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/tables_export/train.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/tables_export/train.png" width="50%"/></a>
<figcaption><br/><i>Train a model to predict ride <code>duration</code>.</i></figcaption>
</figure>

### Export the trained model

Once the model is trained, we'll export the result, so that it can be served from any environment in which you can run a container.  (Note that you could also [deploy][12] your model to the Cloud AI Platform for online prediction).

You'll find the export option under **TEST & USE**.  (See the [documentation][13] for detail on the export process).
You'll need to create a *regional* GCS bucket, in the same region as your model. You also might want to create a sub-folder for the model export in the GCS bucket, so that if you have multiple exports, you can keep track of .  An easy way to create the folder is via the web UI. Here, I've created a `model_export_1` sub-folder.

Click the "Container" card to export your trained model to be run from a Docker container.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/tables_export/export1.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/tables_export/export1.png" width="60%"/></a>
<figcaption><br/><i>Click the "Container" card to export your trained model to be run from a Docker container.</i></figcaption>
</figure>

Browse to select the GCS folder into which you want to export your model, then click the **EXPORT** button.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/tables_export/export2-2.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/tables_export/export2-2.png" width="60%"/></a>
<figcaption><br/><i>Browse to the GCS folder into which you want to export your model.</i></figcaption>
</figure>

When the export is finished, create a local directory to hold your model, e.g. something like `bikes_weather`.

Copy the download command in the cloud console, which will look something like the following:
`gsutil cp -r gs://<your-bucket>/model_export_1//* ./download_dir`

Edit this command: add quotes around the `gs` URI, and remove one of the end slashes.  Then edit `download_dir` to point to the directory you created.  The result should look something like the following. Run it from the parent directory of your `bikes_weather` directory:

```sh
gsutil cp -r 'gs://<your-bucket>/model_export_1/*' ./bikes_weather
```

The exported model will be copied to `./bikes_weather`.

**Note**: to run `gsutil`, you will need [`gcloud`][14] installed. You can run these commands from the [Cloud Shell][15] instead of your local machine if you don't want to install the SDK locally.

## Test your exported model locally

Once you've downloaded your model, you can run and test it locally. This provides a good sanity check before deploying to Cloud Run.
The process is described in the [documentation][16]; we'll summarize here.

- change to the `bikes_weather` directory (or whatever you named it). You should see a `model_export` subdirectory, the result of your download.
- rename the subdirectory as described in the [documentation][17], to remove the timestamp suffix.

Then, create and run a container to serve your new trained model.  Edit the following to point to your renamed directory path:

```sh
docker run -v `pwd`/model-export/tbl/<your_renamed_directory>:/models/default/0000001 -p 8080:8080 -it gcr.io/cloud-automl-tables-public/model_server
```
This starts up a model server to which you can send requests.  Note that we're using the `gcr.io/cloud-automl-tables-public/model_server` container image and mounting our local directory.

Next, [download this `instances.json`][18] file.  If you take a look at it, you can see that it holds data for three prediction instances.
From the directory where you placed `instances.json`, run:

```sh
curl -X POST --data @instances.json http://localhost:8080/predict
```

It may take a second or two for the first request to return, but subsequent requests will be faster.  You’ll get back predictions for all of the instances in the `json` file.
The actual duration for the third instance is 1200.

## View information about your exported model in TensorBoard

You can view your exported custom model in [TensorBoard][19].  This requires a conversion step.
You will need to have TensorFlow 1.14 or 1.15 installed to run the the conversion script.

Then, download [this script][20], e.g. via `curl -O https://raw.githubusercontent.com/amygdala/code-snippets/master/ml/automl/tables/model_export/convert_oss.py`, to the parent directory of `model_export`.  Create a directory for the output (e.g. `converted_export`), then run the script as follows:

```sh
mkdir converted_export
python ./convert_oss.py --saved_model ./model-export/tbl/<your_renamed_directory>/saved_model.pb --output_dir converted_export
```

Then, point TensorBoard to the converted model graph:

```sh
tensorboard --logdir=converted_export
```

You will see a rendering of the model graph, and can pan and zoom to view model sub-graphs in more detail.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/tables_export/tb1.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/tables_export/tb1.png" /></a>
<figcaption><br/><i>You can view an exported custom Tables model in Tensorboard.</i></figcaption>
</figure>

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/tables_export/tb2.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/tables_export/tb2.png" /></a>
<figcaption><br/><i></i></figcaption>
</figure>

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/tables_export/tb3.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/tables_export/tb3.png" /></a>
<figcaption><br/><i>Zooming in to see part of the model graph in more detail.</i></figcaption>
</figure>


## Create a Google Cloud Run service based on your exported model

At this point, we have a trained model that we've exported and tested locally.  Now we are almost ready to deploy it to [Cloud Run][21].
As the last step of prep, we'll create a container image that uses `gcr.io/cloud-automl-tables-public/model_server` as a base image and `ADD`s the model directory, and push that image to the [Google Container Registry][22], so that Cloud Run can access it.

### Build a container to use for Cloud Run

In the same `bikes_weather` directory that holds the `model_export` subdir, create a file called `Dockerfile` that contains the following two lines.  The template is [here][23] as well; **edit the second line to use your correct path to the exported model, the same path that you used above when running locally**.

```
FROM gcr.io/cloud-automl-tables-public/model_server

ADD model-export/tbl/YOUR_RENAMED_DIRECTORY/models/default/0000001
```

Then, build a container from the `Dockerfile`.  In this example we'll call it `bw-serve`.
You can do this as follows (**replace `[PROJECT_ID]` with the id of your project**):

```
docker build . -t gcr.io/[PROJECT_ID]/bw-serve
```

Then push it to the Google Container Registry (again replacing `[PROJECT_ID]` with the id of your project):

```
docker push gcr.io/[PROJECT_ID]/bw-serve
```

(If you get an error, you may need to configure Docker to use gcloud to [authenticate requests to Container Registry][24].)

Alternately, you can use [Cloud Build][25] to build the container instead, as follows:

```
gcloud builds submit --tag gcr.io/[PROJECT_ID]/bw-serve .
```

### Create your Cloud Run service

Now we're ready to deploy the container we built to Cloud Run, where we can scalably serve it for predictions.  Visit the [Cloud Run page in the console][26]. (Click the “START USING..” button if necessary).  Then click the **CREATE SERVICE** button.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/cloud_run1%202.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/cloud_run1%202.png" width="40%"/></a>
<figcaption><br/><i>Creating a Cloud Run Service</i></figcaption>
</figure>

For the container URL, enter the name of the container that you just built above. Select the “Cloud Run (fully managed)” option.  Create a service name (it can be anything you like). Select the **Require Authentication** option.
Then, click on **SHOW OPTIONAL REVISION SETTINGS**.  Change the **Memory allocated** option to **2GiB**.
Leave the rest of the defaults as they are, and click **CREATE**.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/automl/cloud_run2.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/automl/cloud_run2.png" width="50%"/></a>
<figcaption><br/><i>Set your service instances to use 2GiB of memory</i></figcaption>
</figure>

### Send prediction requests to the Cloud Run service

Once your Cloud Run service is deployed, you can send prediction requests to it.  Your new service will have a URL that starts with your service name (and ends with `run.app`). You can send JSON predictions to the Cloud Run service just as with the local server you tested earlier; but with Cloud Run, the service will scale up and down based on demand.

Assuming you selected the **Require Authentication** option, you can make prediction requests like this:

```bash
curl -X POST -H \
"Authorization: Bearer $(gcloud auth print-identity-token)" --data @./instances.json \
https://<your-service-url>/predict
```

(If you set up your Cloud Run service endpoint so that it does not require authentication, you don’t need to include the authorization header in your `curl` request).

## What’s next?

In this post, we walked through how to export a custom AutoML Tables trained model, view model information in TensorBoard, and build a container image that lets you serve the model from any environment.  Then we showed how you can deploy that image to Cloud Run for scalable serving.

Once you’ve built a model-serving container image, it’s easy to deploy it to other environments as well.  For example, if you have installed [Knative serving][27] on a [Kubernetes][28] cluster, you can create a Knative *service* like this, using the same container image (again replacing `[PROJECT_ID]` with your project):

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: bikes-weather
spec:
  template:
    spec:
      containers:
        - image: gcr.io/[PROJECT_ID]/bw-serve
```

(While our example model fits on a 2GiB Cloud Run instance, it’s possible that other of your models may be too large for the managed Cloud Run service, and serving it via Kubernetes/[GKE][29] is a good alternative).

If you’re curious about the details of your custom model, you can use Stackdriver Logging to [view information about your AutoML Tables model][30]. Using Logging, you can see the final model hyperparameters as well as the hyperparameters and object values used during model training and tuning.

You may also be interested in exploring the updated [AutoML Tables client libraries][31], which make it easy for you to [train and use Tables programmatically][32], or reading about how to create a _contextual bandit_ model pipeline [using AutoML Tables, without needing a specialist for tuning or feature engineering][33].

[1]:	https://cloud.google.com/automl-tables/docs/
[2]:	https://cloud.google.com/automl-tables/docs/model-export
[3]:	https://cloud.google.com/run/docs/
[4]:	https://www.tensorflow.org/tensorboard
[5]:	https://cloud.google.com/automl-tables/docs/
[6]:	https://console.cloud.google.com/automl-tables/datasets
[7]:	https://googleapis.dev/python/automl/latest/gapic/v1beta1/tables.html
[8]:	https://console.cloud.google.com/bigquery?p=bigquery-public-data&d=london_bicycles&page=dataset
[9]:	https://console.cloud.google.com/bigquery?p=bigquery-public-data&d=noaa_gsod&page=dataset
[10]:	https://console.cloud.google.com/automl-tables/datasets
[11]:	https://cloud.google.com/automl-tables/docs/problem-types
[12]:	https://cloud.google.com/automl-tables/docs/predict
[13]:	https://cloud.google.com/automl-tables/docs/model-export
[14]:	https://cloud.google.com/sdk/install
[15]:	https://cloud.google.com/shell/
[16]:	https://cloud.google.com/automl-tables/docs/model-export
[17]:	https://cloud.google.com/automl-tables/docs/model-export#run-server
[18]:	https://raw.githubusercontent.com/amygdala/code-snippets/master/ml/automl/tables/model_export/instances.json
[19]:	https://www.tensorflow.org/tensorboard
[20]:	https://github.com/amygdala/code-snippets/blob/master/ml/automl/tables/model_export/convert_oss.py
[21]:	https://cloud.google.com/run/docs/
[22]:	https://cloud.google.com/container-registry/
[23]:	https://raw.githubusercontent.com/amygdala/code-snippets/master/ml/automl/tables/model_export/Dockerfile.template
[24]:	https://cloud.google.com/container-registry/docs/quickstart#add_the_image_to
[25]:	https://cloud.google.com/cloud-build/docs/quickstart-docker
[26]:	https://console.cloud.google.com/marketplace/details/google-cloud-platform/cloud-run
[27]:	https://github.com/knative/serving
[28]:	https://kubernetes.io/
[29]:	https://cloud.google.com/kubernetes-engine/
[30]:	https://cloud.google.com/automl-tables/docs/logging
[31]:	https://googleapis.dev/python/automl/latest/gapic/v1beta1/tables.html
[32]:	https://github.com/GoogleCloudPlatform/python-docs-samples/tree/master/tables/automl/notebooks
[33]:	https://cloud.google.com/blog/products/ai-machine-learning/how-to-build-better-contextual-bandits-machine-learning-models
