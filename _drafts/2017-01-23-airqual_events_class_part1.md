---
layout: post
title:  "Applying Machine Learning to Air Quality Events Analysis (Part I)"
date:   2017-01-23 15:00:00
categories: "machine learning"
comments: true
author: Antoine Galataud
---

<style type="text/css">
.callout {
    float: right;
    margin-left: 5px;
    width: 200px;
}
</style>

![orion galaxy?]({{ site.url }}/assets/airqual_events_class/orion_nebula.jpg)

<br/>

Why was Dark Vador wearing a mask? When you hear him breathing, it's obvious he had serious lung problems, at least severe asthma. Maybe he didn't pay attention to Death Star indoor air quality enough. 
But who could blame? This was a huge space to monitor, with troopers coming and going all the time, cosmic dust from all galaxies, chemicals and exhaust gazes from spaceships... 

Indoor air quality can harm your internals, we know that. But if we aren't able to know pollution sources, it gets tricky to avoid them; beyond taking basic actions when air quality degrades, it'd much more interesting to remove the source, change specific bad habits, automatically trigger the appropriate solution. It's useful to trigger your ventilation every time VOC level is above a threshold, but what if you become aware that it comes from your house cleaning habbits?

This is one of the subjects we're working on at Airboxlab, and this article will be the first of a series of 2 describing how we decided to tackle the problem of pollution source awareness and the benefits it could bring. It also relates the mistakes we did along the way, and how we learned from them.

## First approach: experiments and lessons learned

In the early days we were very enthusiastic: we'd have to monitor 1 or 2 houses for a while, ask users to note what they were doing (cooking, vacuum cleaning, ...) while we were monitoring. Even better, we could get super accurate insight by installing extra sensors and monitors. 

When the experiment ended, we collected data and worked in collaboration with the [LIST](http://list.lu) to build our models. After a few weeks of work, conclusions came like a cold shower: no way to extract any significant pattern from collected data. Although graphs showed correlation between user activity and sensors response, no model could be built. Attempts to build classification models showed disappointing accuracy.

However, this wasn't a show stopper. We learned a lot from this first failed attempt. 

- although accuracy was good and users carefully reported time there were doing things that could impact air quality, experiment was too short in time and data was scarse
- dataset was imbalanced: some classes we wanted to learn about had no data at all
- conducting experiment with different sensors, in a controlled environment, isn't a portable approach. Our users wouldn't be equipped the same way, and they obviously wouldn't be that disciplined in reporting their actions. The resulting model wouldn't match our users habits

More importantly, we realized from last point that users were doing what we asked them. But our real users, the ones who bought our product and installed the app, would they try that hard to help us collect data? At that time, we weren't able to ask them about causes because we weren't telling them when *something* happened. Solution at that time was to open the app, select a time range and select a catagory. Actually, some users were already doing it. But without any trigger, it was very hard to have a clue about what happened, when exactly, for how long, etc. Existing tags dataset collected from real users input was inaccurate and improper to learning. 

It became clear initial problem could be divided into 2 distinct ones:

- how can we detect air quality events?
- how can we classify them?

And that would mean not a single model or pipeline to build, but several. First things first, we started to work on air quality events detection so we could notify users and ask them to tag air quality events.

## Air quality events: an (approximate) definition 

This first unsuccessful attempt to give meaning to Foobot readings lead us to fundamental questions we eluded from the beginning: what do we want to detect? Air quality events. Ok, but what is it?

Air quality event definition isn't as simple as it sounds. One may think of all pollution events that can happen at home, like when you burn your cooking, that will generally translates into a sudden increase of particulate matters (PMs).

But events aren't only about pollution, and we wanted to be able to detect more to get. What about opening a window? At first sight, it should be revealed by a change in temperature and/or humidity. But in what direction (increase or decrease)? In what proportion? 
This kind of event is also depending on external factors like season and location: in winter in New York, opening the window will have a different event signature than during summer, or in a different location like countryside. 

Also, events can be defined by different features, and not only by a sensor value at a specific moment of time: volatility of the value, difference between the minimum and maximum value over a period of time, necessary time to revert to initial value...

More over, let's think about sensor value context: a small change in one sensor value on a short period of time may not be significant when seen alone. But coupled with another small change from another sensor, this could get interesting. Even more if all sensors are impacted. And direction also has its role to play (increase vs decrease): a combination of increases and/or decreases may be frequent and not seen as an event, but another combination could be more rare, and worth paying attention to it.

Let's have a look at below graph: it's 24 hours in the day of a Foobot. Lot of things happen and it illustrates well the complexity of detecting events, without generating too much noise for the user (not every variation is an event). Some events are brutal changes of values in a short period of time, some last longer and there is a bigger period of time between start and end phases. Some involves all sensors but the 3 last are only reflecting on temperature and humidity.

![sensors graph]({{ site.url }}/assets/airqual_events_class/sensors_values_events.png)

Finally, air quality event detection is a non trivial problem. It requires more than simple algorithm or heuristics: this is a very good application of using machine learning algorithms.

## Air Quality Events Detection

### Oddities are in the air

Air quality events we want to detect are actually abnormal and most probably infrequent changes in the values sent by Foobot sensors. For instance, when you open the oven, important amount of PMs can be released (if something burnt), or when you open your window, temperature and humidity may change, as well as VOCs, CO2 and PMs.

We took the following approach: we know we don't know all combinations of the events we want to detect, and we don't even know the events we want to detect. Actually, we want to detect and classify much more than a few categories, and the best way is to let users tell us what they do.

We also want to let users know with no delay that something happen and ask them to tag the event, providing a list of possible categories, but also a free text field so they can create new categories. 2 benefits to this approach:

- users want to be able to react immediately when air quality degrades 
- reducing time between detection and notification gives users more chances to remember what they were doing at that time. Sending notifications or emails every day in batch wouldn't be efficient.
     
To achieve this, we needed to design and implement 2 technical parts:

- something able to detect anomalies by comparing a vector of values to an existing and already organized set of vectors
- a streaming layer to continuously analyze incoming data

### Machine learning to the rescue

[*Anomaly detection*](https://en.wikipedia.org/wiki/Anomaly_detection) (or [*outlier detection*](https://en.wikipedia.org/wiki/Outlier#Detection)) is a common problem in data-mining.
The concept of anomaly detection is quite general, but the idea is to detect outliers (observations very distant to others) from a dataset, and that can be used in a variety of areas like hardware failure detection, intrusion detection systems, or fraud detection. 

There are also different ways to tackle the problem, with a complexity ranging from basic to very complex (neural networks).

We decided to head into [K-Means](https://en.wikipedia.org/wiki/K-means_clustering) which is a clustering technique (unsupervised learning): given a number *k*, the algorithm will divide the dataset into *k* groups (or *clusters*) in a way that minimizes distance between cluster center and data points. K-Means is also relatively fast to train, and its output is easy to understand (compared to other algorithms). 

When applied to the entire dataset, K-Means training will output a model composed of *centroids*: basically coordinates of cluster centers. Centroids are computed to minimize distance between themselves and surrounding data points.

Parameter *k* is left to the discretion of the data scientist: depending on clustering task needs, you may want more or less cluster centers. If minimizing distance is your objective, it's possible to iteratively train models while incrementing *k*, and compute average distance of each data point to its centroid for each run: after a specific value of *k*, accuracy will stop increasing (plateau).  On the contrary, if you want to rough outline patterns in your data, you may want to keep *k* low.

### Anomaly detection with K-Means

Let's dig into anomaly detection, as we talked about K-Means basics, but it's not yet clear at this point how it can help to detect anomalies. 

It's quite simple: every time a data point will be classified with our trained model, the output will be the coordinates of the closest centroid. We can then compute [euclidean distance](https://en.wikipedia.org/wiki/Euclidean_distance) between data point vector and centroid vector to see how far it is from the cluster center. Anomaly detection will make us of this distance: if data point is far, it basically means it's something we didn't see very often in the training dataset.

Going a step further, from the model we trained we'll only select centroids for clusters that classified the more entries. If we take a basic example of 5 clusters, initial training could output something like below:

~~~ 
1: (0.1, 0.2, 0.1, 0.5)    => 100000 entries
2: (-0.2, 0.1, -0.05, 0.1) => 50000
3: ...
5: (4, -2, 4, 4)           => 100
~~~ 

The more entries are classified in a cluster, the more frequent they appear in the dataset. Differently said, clusters with more entries represent the more frequent patterns in the data set.

With a 3D representation, it gets clearer how outliers or anomalies will stand out of the crowd: a pack of points reprensenting clusters with most values (below clusters are grouped but you could get groups with a distance between each of them), and suddenly an anomaly, classified in one of the existing clusters but far from the center.

![clustering]({{ site.url }}/assets/airqual_events_class/clustering_outlier.png)

To wrap this up, we're using anamoly detection technique based on K-Means, but we cut the model from its less populated clusters. We'll then use distance computation between vector to classify and its centroid to decide if it's an event or not.

### Events and Datapoints

Ok but then, how do we feed the trainer? This is part of the *feature selection* process, that makes us pick some values or derivatives of values (like aggregations) but not others from the data set, in order to train the model. 

Being specific here, we're interested in variations of sensor values over a period of time. Intuitively, sensor values variation between now and a few minutes ago is what can tell better that something is happening as it reflects a change in the air composition. Absolute values alone are not interesting, because, for instance, you could have indoor air with constant high VOC values. 

Finally, while training models with this kind of data we rapidly realized that clusters with less data had coordinates representing the most important variations in sensors values, while the ones with more data were associated with small variations. Selecting only the latter gave us a clustering model that could outline anomalies.

## Detecting in near real-time

Near real-time detection is where we wanted to go from the beginning. Detecting events with a few hours delay with a batch would have reduced the benefit of it all, as the user couldn't be notified immediately. 

Here, near real-time detection means continuous stream of datapoints that a job transforms and submits to the classifier. In Spark Streaming, it translates into a sliding window that maintains a state for each device. This state is a representation of the variations of sensor values for the device over a period of time. Spark Streaming offers several operators that comes in handy. See [Spark Streaming - Window Operations](http://spark.apache.org/docs/latest/streaming-programming-guide.html#window-operations) for details.
Spark Streaming also allows us to load a K-Means model and to submit vectors to be classified in a continuous manner.

Upon reception, datapoints are transformed and some flags are checked to help make better decisions, like wether or not taking datapoint into account, applying custom transformations, etc. Then we submit the state to the K-Means classifier and depending on distance to the closest cluster, we tag it as an indoor air quality event or not.

Event end is also a matter of clustering: we consider an event as finished if variations between last datapoint received and first event datapoint are classified close to a highly populated cluster centroid.

Sensitivity of event detection and event closing are depending on the configurable threshold that we compare to each distance `euclid_dist(state_vector - centroid)`. In the case of detection, the bigger the threshold is, the less sensitive detection will be. And it's the exact opposite for event closing. It took us quite some time to find *interesting* thresholds: it was all about working with beta testers, checking large amount of detected events, and fine tuning.

When event ends, not only we can easily retrieve datapoints that caused the event to be triggered, but we can also analyse event life: max and min values, variance, duration... All this data will be extremely useful to understand patterns of indoor air quality events.

Below diagram illustrates this flow

![real time detection]({{ site.url }}/assets/airqual_events_class/real_time_detect_diag.png)

### Fooboters own the value

![ingestion]({{ site.url }}/assets/airqual_events_class/foobot_app_notif.png){: .callout} 

Detecting events in a streaming manner has some benefits: our users can be informed immediately, and they can tell us what it is. When an event is detected, it's transformed into a push notification, that users can open in Foobot mobile app, then select in a category he or she thinks was the root cause of the event. This information is then preciously saved, along with event characteristics.

Fooboters have been more than kind with us: not only we got a massive amount of tags, but they also helped us expending our list of categories. As event types list is not closed - users have the ability to give their own tag if they don't find one in the list of choices - we are able to add new event classes, or create sub categories to refine events classification. 

With the help of our users, we collected an important dataset of tags that we're able to join with event characteristics, in order to form another training dataset: the one that will help building an air quality events classification model!

## Conclusion

In this article we've investigated in details some of the internals of indoor air quality event detection at Airboxlab. There is a lot more to say about the different approaches we've tried, the failures we faced, what we've learned.

Detection is non trivial, but we're fairly satisfied with what we obtained. We regularly monitor accuracy of detected events, and it's quite interesting to see amount of events and conditions of classifications. We've also set configurable thresholds that allow to increase or decrease sensitivity of detection. Still, there is room for improvement; for instance, we didn't dig too much into detecting several events happening in the same window of time. This would involve managing competing events and states. Probably interesting to look into, but somewhat difficult to compare to single-state-per-device (current) technique in terms of relevance, accuracy, and user friendlyness.

In our next article we'll discuss the next step in building an air quality event processing pipeline: classification. Of course, collecting all this data reveals its full potential only if we try to find patterns in it, and make our users benefit from our findings. Knowing precisely what's happening in the air of your home can give you precious information for improving it!




