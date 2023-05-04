---
layout: post
title: Exponential Histograms
date: 2023-05-04 19:00 -0400
categories: Histograms
tags:	metrics opentelemetry prometheus histograms
---

In the previous two posts, I have gone over the basics of histograms and summaries and the tradeoffs, benefits, and limitations of each of them. Because they're easy to understand and demonstrate, those posts focused on so-called explicit bucket histograms. The exponential bucket histogram, also referred to as native histogram in Prometheus, is a low-cost, efficient alternative to explicit bucket histograms. In this post, I will go through what they are, how they work, and the problems they solve that explicit bucket histograms struggle with.

# Types of histograms

For the purposes of this blog post, there are two major types of histograms: explicit bucket histograms and exponential bucket histograms. In previous posts, I've focused on what OpenTelemetry calls explicit bucket histograms and Prometheus simply refers to as histograms. As the name implies, an explicit bucket histogram has each bucket configured explicitly by either the user or some default list of buckets.  Exponential histograms work by calculating bucket boundaries using an exponential growth function. This means each consecutive bucket is larger than the previous bucket and ensures a constant relative error for every bucket.

# Exponential histograms

In OpenTelemetry exponential histograms, buckets are calculated automatically from an integer *scale factor*, with larger scale factors offering smaller buckets and greater precision. It is important to select a scale factor that is appropriate for the scale of values you are collecting in order to minimize error, maximize efficiency, and ensure the values being collected fit in a reasonable number of buckets. In the next few sections, I'll go over the scale and error calculations in detail, show you how to select a scale factor, then provide some examples of reasonable scale factors for a few different applications.

# Scale factor

The most important and most fundamental part of an exponential histogram is also one of the trickiest to understand, the scale factor. From the scale factor, bucket boundaries, and by extension resolution, range, and error rates, are derived. The first step is to calculate the histogram base.

The base is a constant derived directly from the scale using the equation `2 ^ (2 ^ -scale)`. For example, given a scale of 3, the base can be calculated as `2^(2^-3) ~= 1.080508`. Because the calculation depends on the power of the negative scale, as the scale grows, the base shrinks and vice versa. As will be shown later, this is the fundamental reason that a greater scale factor results in smaller buckets and a higher resolution histogram.

# Bucket calculation

Given a scale factor and its resulting base, we can calculate every possible bucket in the histogram. From the base, the upper bound of each bucket at index `i` is defined to be `base ^ (i + 1)`, with the first bucket lower boundary of 1. Because of this, the upper boundary of the first bucket at index 0 is also exactly the base. For now, we will only consider nonnegative indices, but negative indexed buckets are also possible and define all buckets between 0 and 1. Keeping with our example using a scale of 3 and resulting base of 1.080508, the third bucket at index 2 has an upper bound of `1.080508^(2+1) = 1.26149`. The following table shows upper bounds for the first 10 buckets of a few different scale factors:

| index | scale -1 | scale 0 | scale 1     | scale 3     |
| ----- | -------- | ------- | ----------- | ----------- |
| n/a   | **1**    | **1**   | **1**       | **1**       |
| 0     | **4**    | 2       | 1.414213562 | 1.090507733 |
| 1     | **16**   | **4**   | 2           | 1.189207115 |
| 2     | 64       | 8       | 2.828427125 | 1.296839555 |
| 3     | 256      | **16**  | **4**       | 1.414213562 |
| 4     | 1024     | 32      | 5.656854249 | 1.542210825 |
| 5     | 4096     | 64      | 8           | 1.681792831 |
| 6     | 16384    | 128     | 11.3137085  | 1.834008086 |
| 7     | 65536    | 256     | 16          | **2**       |
| 8     | 262144   | 512     | 22.627417   | 2.181015465 |
| 9     | 1048576  | 1024    | 32          | 2.37841423  |

I've bolded some of the values here to show an important property of exponential histograms called *perfect subsetting*.

# Perfect subsetting

In the chart above, some of the bucket boundaries are shared between histograms with differing scale factors. In fact, each time the scale factor increases by 1, exactly 1 boundary is inserted between each existing boundary. This feature is called perfect subsetting because each set of boundaries for a given scale factor is a perfect subset of the boundaries for any histogram with a greater scale factor.

Because of this, histograms with differing scale factors can be normalized to whichever has the lesser scale factor by combining neighboring buckets. This means that histograms with different scale factors can still be combined into a single histogram with exactly the precision of the least precise histogram being combined. For example, histogram *A* with scale 3 and histogram *B* with scale 2 can be combined into a single histogram *C* with scale 2 by first summing each pair of neighboring buckets in *A* to form histogram *A'* with scale 2. Then, each bucket in *A'* is summed with the corresponding bucket of the same index in *B* to make *C*.

# Error rate

The error rate of a histogram is defined as the average relative error when estimating the value of a particular-ranked data point. This is important because it is how φ-quantiles are estimated. In a histogram with x data points, the nth percentile is the data point at rank `n / 100 * x`. For an example of this calculation, see my previous post [*Why Histograms*]({% post_url 2023-05-03-why-histograms %}). When estimating ɸ-quantiles it is very important to know the expected relative error rates and maximum relative error rates so that you can effectively monitor your SLOs.

When using linear interpolation, the expected relative error is half the bucket width divided by the bucket midpoint. Because the relative error is the same across all buckets, we can use the first bucket with the upper bound of the base to make the math easy. An example is shown below using a scale of 3.

```
scale = 3
# see above for base calculation
base  = 1.090508

relative error = (bucketWidth / 2) / bucketMidpoint
			   = ((base - 1) / 2) / ((base + 1) / 2)
			   = (base - 1) / (base + 1)
			   = (1.090508 - 1) / (1.090508 + 1)
			   = 0.04329
			   = 4.329%
```

# Choosing a scale

Because increasing the scale factor increases the resolution and decreases the relative error, it may be tempting to choose a large scale factor. After all, why would you want to introduce error? The answer is that there is a positive relationship between the scale factor and the number of buckets required to represent values within a specified range. For example, with 160 buckets (the OpenTelemetry default), histogram *A* with a scale factor of 3 can represent values between 1 and about 1 million; histogram *B* with a scale of 4 the same number of buckets would only be able to represent values between about 1 and about 1000, albeit at half the relative error. To represent the same range of values as *A* with *B*, twice as many buckets are required; in this case 320.

This brings me to the first most important point of choosing a scale, *data contrast*. Data contrast is how you describe the difference in scale between the smallest possible value x and the largest possible value y in your dataset and is calculated as the constant multiple c such that `y = c * x`. For example, if your data is between 1 and 1000 milliseconds, your data contrast is 1000. If your data is between 1 kilobyte and 1 terabyte, your data contrast is 1,000,000,000. Data contrast, scale, and the number of buckets are all interlinked such that if you have 2, you can calculate the third.

Fortunately, if you are using OpenTelemetry, scale choice is largely done for you. In OpenTelemetry, you configure a maximum scale (default 20) and a maximum size (default 160), or number of buckets, in the histogram. The histogram is initially assumed have the maximum scale. As additional data points are added, the histogram will rescale itself down such that the data points always fit within your maximum number of buckets. The default of 160 buckets was chosen by the OpenTelemetry authors to be able to cover typical web requests between 1ms and 10s with less than 5% relative error. If your data has less contrast, your error will be even less.

# Negative or zero values

 For the bulk of this post we have ignored zero and negative values, but negative buckets work much the same way, growing larger as the buckets get further from zero. All of the math and explanation above applies in the same way to negative values, but they should be substituted for their absolute values, and upper bounds for buckets are lower bounds (or upper absolute value bounds). Zero values, or values with an absolute value less than a configurable threshold, go into a special zero bucket. When merging histograms with differing zero thresholds, the larger threshold is taken and any buckets with absolute value upper bounds within the zero threshold are added to the zero bucket and discarded.

# OpenTelemetry and Prometheus

Compatibility between OpenTelemetry and Prometheus is probably a topic large enough for its own post, and I may write that post, but for now it is enough to state that for all practical purposes, OpenTelemetry exponential histograms are 1:1 compatible with Prometheus native histograms. Scale calculations, bucket boundaries, error rates, zero buckets, etc are all the same. For more information, I recommend you watch this talk given by  Ruslan Vovalov and Ganesh Vernekar: [Using OpenTelemetry’s Exponential Histograms in Prometheus](https://www.youtube.com/watch?v=W2_TpDcess8)