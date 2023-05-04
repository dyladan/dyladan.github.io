---
layout: post
title: Histograms vs Summaries
date: 2023-05-03 18:03 -0400
categories: Histograms
tags:	metrics opentelemetry prometheus histograms
---

In many ways, prometheus histograms and summaries appear quite similar.
They both roll up many data points into a data structure for efficient processing, transmission, and storage.
They can also both be used to track arbitrary quantiles such as the median or p99 of your data.
So the question is raised when to use a summary and when to use a histogram.

# Histograms

Since I just published a post about [histograms and when they are useful]({% post_url 2023-05-02-why-histograms %}), I will only provide a quick summary here.
A histogram is a data structure which describes the distribution of a set of data points.
For example, one may collect all response times to an HTTP endpoint and describe them as a histogram with 10 bins ranging from 0 to 1000 milliseconds.
Each bin counts the number of requests that fall within its range.

<svg class="fig-1"></svg>
<script>
  new chartXkcd.Bar(document.querySelector('.fig-1'), {
    title: 'Response Time (1260 requests)',
    xLabel: 'Milliseconds',
    yLabel: 'Requests',
    data: {
      labels: ['< 10', '< 20', '< 30', '< 50', '< 75', '< 100', '< 200', '< 300', '< 500', '< 750', '< 1000', '1001+'],
      datasets: [{
        data: [100, 120, 150, 210, 220, 180, 130, 80, 40, 10, 5, 15],
      }],
    },
    options: { // optional
      yTickCount: 3,
      legendPosition: chartXkcd.config.positionType.upLeft
    }
  });
</script>

From this we can estimate φ-quantiles like the 90th percentile.
We know there are 1260 requests, so the 1134th-ranked (`1260 * .90`) request represents the 90th percentile.
We can then calculate that the request would fall in the 8th bucket (`300 <= x < 500`) by summing the bucket counts until we exceed that rank.
Finally, using relative rank within the bucket of 24 (`1134 - 1110`), we can estimate the p90 value to be 360ms (`300 + ((24 / 80) * (500 - 300))`) using linear interpolation.
It is important to know that this is an *estimation* and could be off by as much as 60ms (`360 - 300`), a relative error of 17% (`60 / 360`).
This error can be mitigated by configuring more and smaller buckets around your SLO values, but never eliminated.

One important property of histograms is that they are *aggregatable*, meaning that as long as the bucket boundaries line up, an arbitrary number of histograms can be combined into a single histogram with no loss of data or precision.
This means that an arbitrary number of hosts can report histogram data structures to a server, which can aggregate and compute quantiles from all of them as if they were reported by a single host.
By collecting histograms from 1 or many hosts over a long period of time, developers can gain a strong understanding of how their data is distributed and how that distribution changes over time.

# Summaries

Summaries work in almost the opposite manner.
When a summary is configured it is given a φ-quantile to track, an acceptable error range, and a decay rate.
For example, a summary may track p99 ± 0.5% with a decay rate of 5 minutes.
The math is more complex so it won't be discussed here, but one important distinction is that the value is calculated on the client before it is sent to the server.
The most important consequence of this is that summaries from multiple clients *cannot be aggregated*.
Another disadvantage is that if you cannot query arbitrary φ-quantiles, only those which you have configured and collected in advance.

Given these disadvantages, summaries do have some advantages.
First, they trade off a small performance penalty on the client for a significant reduction in transmission, storage, and server processing cost.
In our histogram example above, it is represented as 12 separate timeseries: 1 counter for each bucket + 1 bucket for out of range values + a total sum of all values.
That is for a single, relatively modest, histogram with no attributes to multiply cardinality.
By comparison, the summary is only a single timeseries for the precomputed `p99` value.
Second, they have very low and configurable relative error rates.
The in the histogram example above, we had a potential relative error of 17% where our summary is guaranteed to be within ± 0.5% accuracy.

# So which should you choose?

The disappointing answer is "it depends," and there is no one-size-fits-all solution.
If you need to aggregate data from many sources, then histograms may be the right choice.
If you are collecting a large number of separate metrics with very strict SLOs, or your prometheus server is particularly resource constrained, then maybe summaries are the right choice for you.
Maybe your ideal solution is a hybrid with some histograms for flexible querying and some summaries.
Only you can know the ins and outs of your own system and design an observability solution around it that is accurate and flexible.
The key is knowing the strengths and limitations of the data structure you're using so you can make informed decisions.

# Bonus round: native histograms

I'm planning a longer post on this so I'll keep this short, but many of the key disadvantages of histograms are mitigated by native histograms, called exponential histograms in OpenTelemetry.
Available in Prometheus as an experimental feature since v2.40.0, and stable in the OpenTelemetry specification as of v1.17.0, native histograms enable very efficient data collection and transmission, fewer, and a constant number of, timeseries created per histogram, and very low relative error rates.
They achieve these and other benefits by defining buckets automatically according to a scale factor and resizing intelligently as needed.
If you're not happy with the state of your current histograms and summaries, I encourage you to give native histograms a try.
As of this writing there are no official Prometheus docs on native histograms, but if you stay tuned I plan to add a thorough explanation of them in the coming days.

Until then, here are some talks I found helpful:

- [PromCon EU 2022 - Native Histograms in Prometheus - Ganesh Vernekar](https://promcon.io/2022-munich/talks/native-histograms-in-prometheus/)
- [Kubecon EU 2023 - Prometheus Native Histograms in Production - Björn Rabenstein, Grafana Labs](https://www.youtube.com/watch?v=TgINvIK9SYc)
- [Using OpenTelemetry’s Exponential Histograms in Prometheus - Ruslan Kovalov & Ganesh Vernekar](https://www.youtube.com/watch?v=W2_TpDcess8)

edit: mixed up the x and y axis labels on the histogram