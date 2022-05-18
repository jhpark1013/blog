---
title: Real-time metrics using histograms
date: 2022-05-03T14:31:57-04:00
---
Wouldn't it be cool to accurately analyze trillions of data points without taking up hundreds of terabytes of storage? Say for instance, you wanted to quantify real time query latencies within a specified error-bound? This is actually possible and I wanted to write a bit about the underlying principles. There are a few papers (see references at the end of the blog post) by Datadog [^1] and Gil Tene [^2] that describe this in more detail. Although I had been thinking about applying this for analyzing performance metrics on routers, the math behind it is applicable for any applications that require low latency statistical analysis of incoming streams of data in real time.

## 1. Benefits of relative-error histograms
Traditional metrics are counter (aggregate by sum) or gauge (aggregate by last value or average) which are not accurate. We want to be able to answer questions not just about the average or max latency, but we want to know, for instance, the 99th percentile latency. There are several algorithms that are based on percentile aggregations. To name a few: DDsketch (Datadog), HDR histogram (Gil Tene), OpenHistogram (Circonus).

The key things to keep in mind are:
1. accuracy
2. speed: quick insertions and queries
3. cost: bounded-size storage
4. flexibility: mergeability

We should be able to answer the question “how many requests in August 16th 9:12-9:46 were served within 100 ms?” this is surprisingly hard to measure with all three qualities mentioned above. Let’s discuss the criteria in more detail.

### 1.1 Speed and storage
Prior methods would take every single data, sort it and examine the rank. Instead, we want to be able to examine each item in the stream only once and at the same time limit the memory usage. We can keep the memory usage logarithmic to the size of the stream or constant with a fixed upper bound by using percentile-aggregate-based methods suggested by DDsketch and HDR histogram.
They use a very inexpensive data structure in which they have a maximum bound on the number of buckets (N buckets). And they create these buckets dynamically so that they only use the buckets that are needed. If there is any case where they need N+1 buckets, they’d “roll up” the lower buckets (the lower percentiles that are generally not as interesting).
The insertion and query of their data structure are fast. Each insertion is just two operations — find the bucket and increase the count (sometimes there is an allocation). Queries only look at a fixed number of buckets. This is a constant factor that can be in the range of thousands or less.

### 1.2 Accuracy
A simple bucket histogram (where we’re splitting the curve into discrete parts and storing the height or count in a bucket) — this method assumes even distribution and doesn’t tell us the distribution within those buckets.
Algorithms like gk sketch are rank error based. It’ll give you a bounded error on the response of a query of the data, however, we are mostly interested in the relative error guarantee. Say you want accuracy within +/- 1% in rank. If you query for the 99th percentile, you will be guaranteed to get a value between the 98th and 100th percentile, however, this method is a challenge for tail distributions. (And the vast majority of datasets that people want to look at distributions across are really long tail distributions). This rank-error-based method wouldn’t give the real 99th percentile because there is high variability within the 98th and 100th percentile.. although couldn’t you choose the granularity to be smaller than 1%? but this would be expensive because more storage is needed for more buckets.
The algorithm that gives you a relative error guarantee is more accurate (especially in the tail distribution) because the value that we return is guaranteed to be within 1% of the actual value of the 99 percentile latency. This means that we can answer the question “are 99% of the requests <= 500 ms +/- 1%” i.e. 99% of requests are guaranteed <= 505 ms.
(DDSketch has an advantage over HDR histogram in that they have no published guarantees and their range is bounded)

### 1.3 Mergeability
What we’re looking for in this data structure is for new sketches (that could be different in size) to be merged into the original sketch with the same error guarantees.

## 2. Approximations for the distribution metrics

### 2.1 Notable nomenclature
sketch: data structure from stream processing literature

### 2.2 Parameters
The x-axis of the histogram is logarithmic. We mostly care about the 99th percentile and most of the time the distribution of latencies have long tails. So it makes sense to evaluate the tail in log scale.

The sketch divides the curve of distributions into buckets. Each bucket counts the number of values that falls between the bucket boundaries. Merging two sketches with the same distribution shape is as simple as summing up bucket counts by index.

* $$x_{min}$$ smallest possible observed value
* $$x_{max}$$ largest possible observed value
* $$s$$ significant figures ($$1$$ to $$5$$)
* $$\alpha$$ relative error
* $$\gamma^i - \gamma^{i-1}$$ bucket size

I'm not sure about this part:
The distribution is $$\gamma = (1+\alpha)/(1-\alpha)$$

The significant figures controls the relative error induced by mapping observations to histogram bins.
* For $$s=1$$, every observation x should be placed in a bin with boundaries no further than $$10\%$$ from x.
* For $$s=2$$, results should be accurate to $$1\%$$.
* For $$s=3$$, results should be accurate to $$0.1\%$$.

That is, the lower boundary of bin $$i$$ is $$\gamma_{i-1}$$, and should be the minimum possible value for $$x$$.
$$x$$ is contained within $$\gamma_{i-1} < x <= \gamma_{i}$$.
* For $$s=1$$ ($$10\%$$): $$x-0.9x < x < 1.1x$$
* For $$s=2$$ ($$1\%$$): $$x-0.09x < x < 1.01x$$

The relative error $$\alpha$$ for above is $$0.1$$ and $$0.01$$ respectively..

Let's define the bucket parameters which is needed to determine how to assign points to the buckets.
* $$x$$: true value
* $$i$$: bucket number
* $$i_{max}$$: number of buckets
* $$C$$: number of sub_buckets (described in next section)
* $$b(i,j)$$: bin boundary function
* $$r(i,j)=k$$: coordinate mapping function (assigning points to buckets)

### 2.3 Bucketing

HDRHistogram specifies each bin in terms of two indices:

* **bucket** $$i$$ which indexes exponentially over the full range of values.
* **sub-bucket** $$j$$ which indexes linearly within a buckets.

The bin widths grow as the values themselves grow, allowing us to achieve constant relative error.  The bin boundaries are defined as a function of two indices for bucket $$i$$ and sub-bucket $$j$$.

#### 2.3.1 Bin boundaries
The bin boundary is defined as $$b(i,j) = (j+C)(2^i)$$

Where $$C$$ is the number of sub-buckets and is a power of $$2$$, or $$C=2^D$$. This makes the math easier to implement because we can use simple bit shifting to divide out the power of $$2$$ term to calculate the bin index.

#### 2.3.2 Number of bins
Number of sub-buckets is represented by $$C$$. Notice at $$j=C$$, we have a bin boundary. From the bin boundary equation above, we can calculate the number of sub-buckets. Calculate by substituting $$j$$ for $$C$$:

$$
\begin{align}
b(i,j) = (j+C) * (2^i) \\
b(i,C) = (2C) * (2^i) \\
 = C*2^{i+1} \\
 = (0+C)*2^{i+1} \\
 = b(i+1,0)
\end{align}
$$

that is, at $$j=C$$, we roll over to the next bucket $$i+1$$. This means that $$j$$ ranges from $$0$$ to $$C-1$$ and there are $$C$$ sub-buckets per bucket.

How many buckets do we need to cover the range $$x_{min}$$ to $$x_{max}$$?
(Note: $$i$$ here is the bucket number).
If $$x_{min}$$ is $$0$$, we need to find the smallest value of $$i$$ such that $$C * 2^i > x_{max}$$, which can be computed as:

$$i_{max} = ceil(log_2(x_{max}/C))$$

#### 2.3.3 Assigning points to buckets
Let the relative error $$\alpha$$ be a "mask" defined by $$2C-1$$ which is the largest value which can be represented int he first bucket $$i=0$$.

Note: $$2C-1$$ comes from $$b(i,j)$$ at bin boundary **minus $$1$$**.
As mentioned in 2.3.1, when C is a power of $$2$$, substitute $$C$$ in the equation as $$C=2^D$$.

$$
\begin{align}
b(i,C) - 1 = 2C - 1\\
= 2^{D+1} - 1
\end{align}
$$

Next we represent this in code.

Compute the bucket index $$i$$ by identifying the index of the leftmost 1 in the binary representation of $$x \| \alpha$$ which is a bitwise OR of $$x$$ and $$\alpha$$.

The OR operation happens with the value and the sub-bucket mask.

```python
self.sub_bucket_mask = (self.sub_bucket_count - 1) << self.unit_magnitude
```

Subtract $$D+1$$ from this quantity (in the HDRHistogram python code)

```python
def _get_bucket_index(self, value):
    # smallest power of 2 containing value
    pow2ceiling = 64 - self._clz(int(value) | self.sub_bucket_mask)
    return int(pow2ceiling - self.unit_magnitude -
               (self.sub_bucket_half_count_magnitude + 1))
```
clz is defined as:
```python
def _clz(self, value):
    """calculate the leading zeros, equivalent to C __builtin_clzll()
    value in hex:
    value = 1 clz = 63
    value = 2 clz = 62
    value = 4 clz = 61
    value = 1000 clz = 51
    value = 1000000 clz = 39
    """
    return 63 - (len(bin(value)) - 3)
```

A C implementation below..
Mask is defined and the bucket index is found by subtracting the unit_magnitude and sub_bucket_half_count_magnitude.
```c
static void record_histogram_entry(struct histogram *h, u32 v)
{
	int mask = (h->header.subbin_count-1) << h->header.unit_magnitude;
	int bkt_idx = v == 0 ? 0 : fls(v | mask) -
                h->header.unit_magnitude - h->header.subbin_half_count_magnitude + 1;
	int subbin_idx = v >> (bkt_idx + h->header.unit_magnitude);
	int bucket = ((bkt_idx+1) << h->header.subbin_half_count_magnitude) +
                (subbin_idx - h->header.subbin_count/2);

}
```



## References
[^1]: [Datadog](http://www.vldb.org/pvldb/vol12/p2195-masson.pdf)
[^2]: [HDRhistogram](http://hdrhistogram.org/)
