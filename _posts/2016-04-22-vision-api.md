---
layout: post
title:  "Using the Cloud Vision API with Twilio Messaging on Kubernetes"
categories:
- ML
tags:
- vision_api
- machine_learning
date: 2016-04-22
---

The [Google Cloud Vision API](https://cloud.google.com/vision/) has just moved to GA (General Availability) status.

<img src="https://amy-jo.storage.googleapis.com/images/cat_and_laptop.jpg" width="300"/>

The Vision API lets you create applications that can classify images into thousands of categories (e.g., "sailboat", "lion"... or **cat** and **laptop** as above); can detect faces and other objects in images (including predicting "sentiment"); can perform OCR (detection of text in images); can detect landmarks (like the Eiffel Tower); can detect logos; and can moderate for offensive content.
You can [see Jeff Dean demoing the Vision API in this video](https://www.youtube.com/watch?v=ud2Ipnq0pTU).

[This github repo](https://github.com/GoogleCloudPlatform/cloud-vision) has a number of Vision API examples, written in different languages, and showing off different aspects of the Cloud Vision API. Some of the examples are simple scripts, and others are a bit more complex.

A [new example called 'twilio-k8s'](https://github.com/GoogleCloudPlatform/cloud-vision/tree/master/python/twilio/twilio-k8s) has just been added to the repo. It shows
how to run the
["What's That?" app](https://github.com/GoogleCloudPlatform/cloud-vision/tree/master/python/twilio/twilio-labels) (built by [Julia
Ferraioli](http://www.blog.juliaferraioli.com/2016/02/exploring-world-using-vision-twilio.html)), on
[Kubernetes](http://kubernetes.io/).

The app uses [Twilio](https://www.twilio.com) to allow images to be texted to a given number,
then uses the [Cloud Vision API](https://cloud.google.com/vision/) to find labels in the image
(classify what's in the image) and return the detected labels as a reply text.
Because the app is running on Kubernetes, it's easy to **scale up the app** to support a large number
of requests.

Once you've set up the app on your Google Container Engine (or Kubernetes) cluster, and set up your Twilio number, you can text images to get content labelings:

<a href="https://amy-jo.storage.googleapis.com/images/yard.jpg" target="_blank"><img src="https://amy-jo.storage.googleapis.com/images/yard.jpg" width="300"/></a>

It's easy to modify your own version of the app to look for additional information in the image.  I've in fact modifed my version of the app to look for *logos* too. It's fun to see how well it can do with an incomplete view of a logo:

<a href="https://amy-jo.storage.googleapis.com/images/cl_bar.png" target="_blank"><img src="https://amy-jo.storage.googleapis.com/images/cl_bar.png" width="300"/></a>

After you've set up your app, when you're ready to share it, you can scale up the number of servers running on your Kubernetes cluster so that the app stays responsive.  The example's [README](https://github.com/GoogleCloudPlatform/cloud-vision/blob/master/python/twilio/twilio-k8s/README.md) goes into more detail about how to do all of this.

For a relatively short time, you can try it out here: (785) 336-5113.  
This number won't work indefinitely, though :).


