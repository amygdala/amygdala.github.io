---
layout: post
title: "Using Preemptible VMs for Kubeflow Pipelines"
categories:
- Kubeflow
- ML
tags: kubeflow ml pipelines
date: 2019-08-18
---

## Introduction

In this post, we'll show how you can use [preemptible VMs](https://cloud.google.com/kubernetes-engine/docs/how-to/preemptible-vms) when running [Kubeflow Pipelines](https://www.kubeflow.org/docs/pipelines/overview/pipelines-overview/) jobs, to reduce costs. We'll also look at how you can use [Stackdriver Monitoring](https://cloud.google.com/monitoring/docs/) to inspect logs for both current and terminated pipeline operations.


Preemptible VMs are [Compute Engine VM](https://cloud.google.com/compute/docs/instances/) instances that last a maximum of 24 hours and provide no availability guarantees. The pricing of preemptible VMs is lower than that of standard Compute Engine VMs.
With [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/docs/), it is easy to set up a cluster or node pool [that uses preemptible VMs](https://cloud.google.com/kubernetes-engine/docs/how-to/preemptible-vms).
You can set up such a node pool with [GPUs attached to the preemptible instances](https://cloud.google.com/compute/docs/instances/preemptible#preemptible_with_gpu). These work the same as regular GPU-enabled nodes, but the GPUs persist only for the life of the instance.


[Kubeflow](https://www.kubeflow.org/) is an open-source project  dedicated to making deployments of machine learning (ML) workflows on [Kubernetes](https://kubernetes.io) simple, portable and scalable.  [Kubeflow Pipelines](https://www.kubeflow.org/docs/pipelines/overview/pipelines-overview/) is a platform for building and deploying portable, scalable machine learning (ML) workflows based on Docker containers.


If you're running Kubeflow on GKE, it is now easy to [define and run Kubeflow Pipelines](https://www.kubeflow.org/docs/pipelines/sdk/gcp/preemptible/) in which one or more pipeline steps (components) run on preemptible nodes, reducing the cost of running a job.  For use of preemptible VMs to give correct results, the steps that you identify as preemptible should either be [*idempotent*](https://en.wikipedia.org/wiki/Idempotence) (that is, if you run a step multiple times, it will have the same result), or should checkpoint work so that the step can pick up where it left off if interrupted.

For example, a copy of a [Google Cloud Storage (GCS)](https://cloud.google.com/storage/) directory will have the same results if it's interrupted and run again (assuming the source directory is unchanging).
An operation to train a machine learning (ML) model (e.g., a TensorFlow training run) will typically be set up to checkpoint periodically, so if the training is interrupted by preemption, it can just pick up where it left off when the step is restarted.  Most ML frameworks make it easy to support checkpointing, and so if your Kubeflow Pipeline includes model training steps, these are a great candidate for running on preemptible GPU-enabled nodes.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/kf-pls/preempt2%202.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/kf-pls/preempt2%202.png" width="70%"/></a>
<figcaption><br/><i>A Kubeflow Pipelines job that is using preemptible VMs for its training. When a training step is terminated due to preemption, it is restarted, and picks up where it left off using checkpoint information.</i></figcaption>
</figure>

When you're running a pipeline and a cluster node is preempted, any [pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) running on that node will be terminated as well. You'll often want to look at the logs from these terminated pods. Cloud Platform's [Stackdriver logging](https://cloud.google.com/monitoring/docs/) makes it easy to do this.

In this post, we'll look at how to use preemptible VMs with Kubeflow Pipelines, and how to inspect pipeline steps using Stackdriver.

## Set up a preemptible GPU node pool in your GKE cluster

You can set up a preemptible, GPU-enabled node pool for your cluster by running a command similar to the following, editing the following command with your cluster name and zone, and adjusting the accelerator type and count according to your requirements. As shown below, you can also define the node pool to autoscale based on current workloads.

```sh
gcloud container node-pools create preemptible-gpu-pool \
    --cluster=<your-cluster-name> \
    --zone <your-cluster-zone> \
    --enable-autoscaling --max-nodes=4 --min-nodes=0 \
    --machine-type n1-highmem-8 \
    --preemptible \
    --node-taints=preemptible=true:NoSchedule \
    --scopes cloud-platform --verbosity error \
    --accelerator=type=nvidia-tesla-k80,count=4
```

You can also set up the node pool via the [Cloud Console](https://console.cloud.google.com).

> **Note**: You may need to increase your [GPU quota](xxx) before running this command.


## Defining a Kubeflow Pipeline that uses the preemptible GKE nodes

When you're defining a Kubeflow Pipeline, you can indicate that a given step should run on a preemptible node by modifying the op like this:

```python
your_pipelines_op.apply(gcp.use_preemptible_nodepool())
```

See the [documentation](https://www.kubeflow.org/docs/pipelines/sdk/gcp/preemptible/) for details— if you changed the node taint from the above when creating the node pool, pass the same node toleration to the `use_preemptible_nodepool()` call.

You'll presumably also want to retry the step some number of times if the node is preempted. You can do this as follows— here, we're specifying 5 retries.  This annotation also specifies that the op should run on a node with 4 GPUs available.

```python
your_pipelines_op.set_gpu_limit(4).apply(gcp.use_preemptible_nodepool()).set_retry(5)
```


## An example: making model training cheaper with preemptible nodes

Let's look at a concrete example.
The Kubeflow Pipeline from [this codelab](https://codelabs.developers.google.com/codelabs/cloud-kubeflow-pipelines-gis/#0) is a good candidate for using preemptible steps.  This pipeline trains a [Tensor2Tensor](https://github.com/tensorflow/tensor2tensor/) model on GitHub issue data, learning to predict issue titles from issue bodies, deploys the trained model for serving, and then deploys a webapp to get predictions from the model. It has a uses a TensorFlow model architecture that requires GPUs for reasonable performance, and— depending upon configuration— can run for quite a long time.
I thought it would be useful to modify this pipeline to make use of the new support for preemptible VMs for the training. ([Here](https://github.com/kubeflow/examples/blob/master/github_issue_summarization/pipelines/example_pipelines/gh_summ.py) is the original pipeline definition.)

To do this, I first needed to refactor the pipeline, to separate a GCS copy action from the model training activity. In the original version of this pipeline, a copy of an initial TensorFlow checkpoint directory to a working directory was bundled with the training.
This refactoring was required for correctness: if the training is preempted and needs to be restarted, we don't want to wipe out the current checkpoint files with the initial ones.

While I was at it, I created [**reusable component** specifications](https://www.kubeflow.org/docs/pipelines/sdk/component-development/) for the two GCS copy and TensorFlow training pipeline steps, rather than defining these pipeline steps as part of the pipeline definition.  A reusable component is a pre-implemented standalone [component](https://www.kubeflow.org/docs/pipelines/concepts/component/) that is easy to add as a step in any pipeline, and makes the pipeline definition simpler. The component specification makes it easy to add [static type checking](https://www.kubeflow.org/docs/pipelines/sdk/static-type-checking/) of inputs and outputs as well.
You can see these component definition files [here](https://github.com/amygdala/kubeflow-examples/blob/preempt/github_issue_summarization/pipelines/components/t2t/datacopy_component.yaml) and [here](https://github.com/amygdala/kubeflow-examples/blob/preempt/github_issue_summarization/pipelines/components/t2t/train_component.yaml).

Here is the relevant part of the new pipeline definition:

```python
import kfp.dsl as dsl
import kfp.gcp as gcp
import kfp.components as comp

...

copydata_op = comp.load_component_from_url(
  'https://raw.githubusercontent.com/amygdala/kubeflow-examples/preempt/github_issue_summarization/pipelines/components/t2t/datacopy_component.yaml'
  )

train_op = comp.load_component_from_url(
  'https://raw.githubusercontent.com/amygdala/kubeflow-examples/preempt/github_issue_summarization/pipelines/components/t2t/train_component.yaml'
  )

@dsl.pipeline(
  name='Github issue summarization',
  description='Demonstrate Tensor2Tensor-based training and TF-Serving'
)
def gh_summ(
  train_steps=2019300,
  project='YOUR_PROJECT_HERE',
  github_token='YOUR_GITHUB_TOKEN_HERE',
  working_dir='YOUR_GCS_DIR_HERE',
  checkpoint_dir='gs://aju-dev-demos-codelabs/kubecon/model_output_tbase.bak2019000',
  deploy_webapp='true',
  data_dir='gs://aju-dev-demos-codelabs/kubecon/t2t_data_gh_all/'
  ):

  copydata = copydata_op(
    working_dir=working_dir,
    data_dir=data_dir,
    checkpoint_dir=checkpoint_dir,
    model_dir='%s/%s/model_output' % (working_dir, '{{workflow.name}}'),
    action=COPY_ACTION
    ).apply(gcp.use_gcp_secret('user-gcp-sa'))

  train = train_op(
    working_dir=working_dir,
    data_dir=data_dir,
    checkpoint_dir=checkpoint_dir,
    model_dir='%s/%s/model_output' % (working_dir, '{{workflow.name}}'),
    action=TRAIN_ACTION, train_steps=train_steps,
    deploy_webapp=deploy_webapp
    ).apply(gcp.use_gcp_secret('user-gcp-sa'))

  train.after(copydata)
  train.set_gpu_limit(4).apply(gcp.use_preemptible_nodepool()).set_retry(5)
  train.set_memory_limit('48G')

  ...

```

I've defined the `copydata` and `train` steps using the component definitions, in this case loaded from URLs. (While not shown here, a Github-based component URL can include a specific git commit hash, thus supporting component version control— [here's an example of that](https://github.com/kubeflow/pipelines/blob/master/components/gcp/ml_engine/train/sample.ipynb).)

I've annotated the training op to run on a preemptible GPU-enabled node, and to retry 5 times. (For a long training job, you'd want to increase that number).
You can see the full pipeline definition [here](https://github.com/amygdala/kubeflow-examples/blob/preempt/github_issue_summarization/pipelines/example_pipelines/gh_summ_preempt.py).

(As a side note: it would have worked fine to use preemptible VMs for the copy step too, since if it is interrupted, it can be rerun without changing the result.)

## Preemptible pipelines in action

When the pipeline above is run, its training step may be preempted and restarted.  If this happens, it will look like this in the Kubeflow Pipelines dashboard UI:

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/kf-pls/preempt1-2%202.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/kf-pls/preempt1-2%202.png" width="90%"/></a>
<figcaption><br/><i>A pipeline with a preemptible training step that has been restarted two times.</i></figcaption>
</figure>

The restarted training step picks up where it left off, using its most recent saved checkpoint.  In this screenshot, we're using the Pipelines UI to look at the logs for the *running* training pod, `train(2)`.

If we want to look at the logs for a pod *terminated* by node preemption, we can do this using [Stackdriver](https://cloud.google.com/monitoring/).

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/kf-pls/stackdriver2-2%202.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/kf-pls/stackdriver2-2%202.png" width="90%"/></a>
<figcaption><br/><i>If the pod for a pipeline step has been deleted, a link is provided to look at the logs in Stackdriver.</i></figcaption>
</figure>

Clicking the stackdriver link opens a window that brings up the Stackdriver Log Viewer in the Cloud Console, and sets a *filter* that selects the output for that pod.

<figure>
<a href="https://storage.googleapis.com/amy-jo/images/kf-pls/stackdriver1-2%202.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/kf-pls/stackdriver1-2%202.png" width="90%"/></a>
<figcaption><br/><i>Clicking the 'Stackdriver' link in the Kubeflow Pipelines UI takes you to the relevant logs in the Cloud Console.</i></figcaption>
</figure>

At some later point, the training run completes— in the figure below, after three premptions and retries— and the remainder of the pipeline runs.
<figure>
<a href="https://storage.googleapis.com/amy-jo/images/kf-pls/preempt2%202.png" target="_blank"><img src="https://storage.googleapis.com/amy-jo/images/kf-pls/preempt2%202.png" width="90%"/></a>
<figcaption><br/><i>The full pipeline run, with training completed after 3 preemptions.</i></figcaption>
</figure>


### What's next?

In this post, I showed how you can use preemptible VMs for your Kubeflow Pipelines jobs in order to reduce costs.

To learn more about Kubeflow, including Kubeflow Pipelines, and to try it out yourself, the [Kubeflow documentation](https://www.kubeflow.org/docs/) and [examples repo](https://github.com/kubeflow/examples) are good starting points.
You might also be interested in this recent [Kubeflow Community meeting presentation](https://www.youtube.com/watch?v=fiFk5FB7il8) on what's new in the Kubeflow 0.6 release.















































































