---
layout: post
title:  "Why Histograms?"
date:   2023-05-02 12:00:00
categories: Histograms
tags:	metrics opentelemetry
# cover:  "/assets/instacode.png"
---

A histogram is a multi-value counter which summarizes the distribution of one or more data points.
For example, a histogram may have 3 counters which count the occurrences of negative, positive, and zero values respectively.
Given a series of numbers, `3`, `-9`, `7`, `6`, `0`, and `-1`, the histogram would count `2` negative, `1` zero, and `3` positive values.
A single histogram data point is most commonly represented as a bar chart.

<svg class="histogram-point"></svg>
<script>
  const histPoint = document.querySelector('.histogram-point')

  new chartXkcd.Bar(histPoint, {
    title: 'Positivity of numbers', // optional
    data: {
      labels: ['<0', '0', '>0'],
      datasets: [{
        data: [2, 1, 3],
      }],
    },
    options: { // optional
      yTickCount: 3,
      legendPosition: chartXkcd.config.positionType.upLeft
    }
  });
</script>

This simple example has only 3 possible output values, but it is not uncommon to have many more in a single histogram.
Similarly, this shows only 1 histogram data point, but in the world of observability data is usually constantly exported.
For example, an application may export a histogram every minute which summarizes a metric for the previous minute.
In this way, you can study how the distribution of your data changes over time.

# What are histograms for?

There are many uses for histograms, but their power comes from the ability to efficiently answer queries about the distribution of your data.
These queries most commonly come in some form like "what was the median response time in the last minute?"
These are known as φ-quantiles, and often are abbreviated in a shorthand like `p50` for the 50th percentile or 0.5-quantile, also known as the median.
More generally, the φ-quantile is the observation value that ranks at number φ*N among the N observations.

# Why is that useful?

The most common use-case for histograms in the observability space is defining service level objectives (SLOs).
One example of such an SLO might be ">=99% of all queries should respond in less than 30ms," or "90% of all page loads should become interactive within 100ms of first paint."

In the following chart, you can see the `p50`, `p90`, and `p99` response times plotted for some requests over some time.
From the data, you can see that 50% of requests are served in around 20-30ms or less, 90% of requests are served in under about 80ms, and 99% of requests are served in under around 90ms.
You can very quickly see that at least 50% of your users are receiving very fast response times, but almost all of your users are experiencing response times under 90ms.

<svg class="hist-lines"></svg>
<script>
const histLines = document.querySelector('.hist-lines')
new chartXkcd.Line(histLines, {
    title: 'Response times', // optional
    xLabel: 'Time',
    yLabel: 'Milliseconds',
    data: {
        labels: ['t0', 't1', 't2', 't3', 't4', 't5', 't6', 't7', 't8', 't9', 't10'],
        datasets: [{
            label: 'p50',
            data: [25, 22, 24, 24, 25, 28, 23, 21, 21, 28, 23, 0],
        }, {
            label: 'p90',
            data: [75,72,69,78,81,62,72,82,73,75,71,72],
        }, {
            label: 'p99',
            data: [85,82,89,88,84,72,82,88,83,85,81,77],
        }],
    },
    options: { // optional
        legendPosition: chartXkcd.config.positionType.upLeft
    }
});
</script>

# Other metric types

Another solution might be to define the SLOs you're interested in and collect them as non-histogram metrics in advance as gauges or counters.
This approach works, but requires defining your SLOs before you have an understanding of your data distribution and requires non-trivial implementation at collection time.
It is also inflexible; if you decide to change your SLO from 90% of requests to 99% of requests, you have to make and release code changes, then wait for the old data to age out and the new metric to collect enough data to make useful queries.
Because histograms model data as a distribution from start to finish, they enable you to simply change your queries and get answers on the data you've already collected.
Particularly with exponential histograms, arbitrary distribution queries can be made with very low relative error rates and minimal resource consumption on both the client and the analysis backend.

The inflexibility of this approach also impacts your ability to gauge impact when your SLO is violated.
For example, imagine you are collecting a gauge which calculates the `p99` of some metric and you define an SLO based on it.
When your SLO is violated and an alert is triggered, how do you know it is really only affecting 1% of queries, 10%, or 50%?
A histogram would allow you to answer that question by simply querying the percentiles you're interested in.
You could collect additional gauges for each percentile, but then you've just force users to reimplement histograms on their own.
Probably poorly.

# Other data sources and metric types

You may ask why you would report a separate metric rather than calculating these metrics from your existing log and trace data?
While it is true that for *some* use cases, like response times, this may be possible, it is not possible for *all* use cases.
Even when percentiles can be calculated from existing data, this typically requires querying a massive amount of data to answer a very simple question.
Further, if you are sampling your tracing and logs data, you may be missing crucial information required to asses whether or not you are meeting your SLOs.
