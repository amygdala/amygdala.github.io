---
layout: post
title: "Remote deployment of Kubeflow Pipelines"
categories:
- Kubeflow
- ML
tags: kubeflow cloud_ml kubeflow_pipelines
date: 2019-08-22
---

If you've used [Kubeflow](https://www.kubeflow.org/docs/), you may know that its [Jupyter notebooks](https://www.kubeflow.org/docs/components/jupyter/) installation makes it easy to deploy a [Kubeflow Pipeline](https://www.kubeflow.org/docs/pipelines/) directly from a notebook, using the [Pipelines SDK](https://www.kubeflow.org/docs/pipelines/sdk/).
[This notebook](https://github.com/kubeflow/pipelines/blob/master/samples/core/lightweight_component/Lightweight%20Python%20components%20-%20basics.ipynb), and [other samples](https://github.com/kubeflow/pipelines/tree/master/samples), show examples of how you can do that.
The [Pipelines Dashboard](https://www.kubeflow.org/docs/pipelines/pipelines-quickstart/), which is part of the Kubeflow Dashboard, makes it easy to upload and run pipelines as well.

Sometimes, though, you want to deploy a Kubeflow Pipeline *remotely*, from outside the Kubeflow cluster. Maybe you want to do this via execution of a command-line script on your local laptop; or from a VM that's outside the Kubeflow cluster (like the [AI Platform notebooks](https://pantheon.corp.google.com/mlengine/notebooks/)); or from services like [Cloud Run](https://cloud.google.com/run/docs/), or [Cloud Functions](https://cloud.google.com/functions/docs/) (which let you support event-triggered pipelines until that feature is available natively).

Remote deployment works well, but to implement it, you need to do a bit more setup to create the client connection to Kubeflow Pipelines than you would if you were connecting from within the same Kubeflow cluster. 

For example, the [Kubeflow 'click-to-deploy' web app](https://deploy.kubeflow.cloud/#/deploy), which makes it easy to install Kubeflow on GKE, includes the option to set up a [Cloud Identity-Aware Proxy (IAP)-enabled cluster](https://cloud.google.com/iap/).  Connecting remotely to an IAP-enabled cluster requires configuring and using a
*service account* with the correct permissions.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/kf-pls/kf06_c2d.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/kf-pls/kf06_c2d.png" width="500"/></a>
<figcaption><br/><i>Creating an IAP-enabled Kubeflow installation on GKE using the click-to-deploy web app.</i></figcaption>
</figure>

<p></p>

I've created a series of notebooks that walk through how to do the necessary setup for remote deployment in three different contexts:

- [kfp_remote_deploy-IAP.ipynb](https://github.com/kubeflow/examples/blob/cookbook/cookbook/pipelines/notebooks/kfp_remote_deploy-IAP.ipynb) shows how to do remote Pipelines deployment to an IAP-enabled Kubeflow installation (on GKE).

- [gcf_kfp_trigger.ipynb](https://github.com/kubeflow/examples/blob/cookbook/cookbook/pipelines/notebooks/gcf_kfp_trigger.ipynb) gives an example of how you can use
[GCF (Cloud Functions)](https://cloud.google.com/functions/) to support event-triggering of a Pipeline deployment.
The example shows the GCF function being triggered by the addition or update of a [GCS (Google Cloud Storage)](https://cloud.google.com/storage) object, but there are many other [GCF triggers](https://cloud.google.com/functions/docs/calling/) that you can use.

- [kfp_remote_deploy-port-forward.ipynb](https://github.com/kubeflow/examples/blob/cookbook/cookbook/pipelines/notebooks/kfp_remote_deploy-port-forward.ipynb) walks through how to connect via port-forwarding to the cluster.

**Note**: port-forwarding is discouraged in a production context.

## Using Cloud Function triggers

The use of Cloud Functions to trigger a pipeline deployment opens up many possibilities for supporting event-triggered pipelines.
For example, you might want to automatically kick off an ML training pipeline run once an AI Platform Data Labeling Service
["export"](https://cloud.google.com/data-labeling/docs/export#datalabel-example-python) finishes. There are multiple ways that you could use GCF to do this. You probably don't want to set up a GCF trigger on the export GCS bucket itself, as that would trigger too many times. 

However, you could write a GCS file to a 'trigger bucket' upon completion of the export process, that contains information about the path of the exported files.  A GCF function defined to trigger on that bucket could read the file contents and use the info about the export path as a param when calling `run_pipeline()`.

The  [gcf_kfp_trigger.ipynb](https://github.com/kubeflow/examples/blob/cookbook/cookbook/pipelines/notebooks/gcf_kfp_trigger.ipynb) notebook includes an example of how you could set up something along these lines.


## Summary

In this article, I talked about several different ways that you can access the Pipelines API, and remotely deploy pipelines, from outside your Kubeflow cluster-- including via AI Platform Notebooks, Google Cloud Functions, and running on your local machine. Google Cloud Functions provides a straightforward foundation for supporting many types of event-triggered pipelines, and the GCF notebook shows an example of one way that you could automatically launch a pipeline run on new data after doing an export of Data Labeling Service results.

Give these notebooks a try, and let us know what you think!

