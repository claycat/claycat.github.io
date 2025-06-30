---
title: "Performance Improvement of Optimistic Locking Using Random Backoff"
date: 2024-05-28 12:00:00 +/-TTTT
categories: [Tiketeer]
comments: true
tags: [java, tiketeer]
---

# Introduction

<img src="/images/../../assets/images/2024-06-15-20-13-33.png" alt="Description" style="display:block; width:400px; margin-left:auto; margin-right:auto;">

While implementing a ticketing system using optimistic locking,

one of the performance improvement measures applied was random backoff.

You can find information and background on optimistic locking [here](https://www.notion.so/4975208574964df8beaefc6a753ad837?pvs=21).

To use optimistic locking effectively, developers must catch exceptions themselves in the application

and perform retries.

In this article, we will look at how adding randomness to the retry strategy can yield performance improvements.

# Main Content

In the [previous implementation](https://www.notion.so/4975208574964df8beaefc6a753ad837?pvs=21), when concurrency issues occurred, all failed clients would retry immediately.

<img src="/images/../../assets/images/2024-06-15-20-14-08.png" alt="Description" style="display:block; width:500px; margin-left:auto; margin-right:auto;">

If implemented this way, all clients retry at the same interval, causing load to concentrate at specific moments.

To prevent this, clients should not retry at the same fixed interval after failure but should retry at irregular intervals.

This “irregular interval” is called jitter.

(Besides jitter, there are many retry strategies like exponential backoff, exponential jitter backoff, etc., but here we will mainly discuss linear jitter.)

Let’s see how to add randomness to the retry intervals in the existing code.

<img src="/images/../../assets/images/2024-06-15-20-14-14.png" alt="Description" style="display:block; width:700px; margin-left:auto; margin-right:auto;">

The `@Retryable` annotation takes backoff values as parameters, and you can adjust them using delayExpression in SpeL (Spring Expression Language).

> delayExpression: expression to determine the delay value  
> maxDelay: maximum delay value  
> multiplier: multiplier by which the delay increases (useful for exponential backoff)

Using this expression, the delay value is determined randomly between 300 and 500 each time.

## Measurement

<img src="/images/../../assets/images/2024-06-15-20-14-35.png" alt="Description" style="display:block; width:1000px; margin-left:auto; margin-right:auto;">

Left: random, Right: fixed

To compare the performance of random intervals and fixed intervals, we wrote the code above and conducted load testing.

We used K6, and you can find the load script [here](https://github.com/Tiketeer/Tiketeer-BE/blob/develop/stress-test/k6-scripts/stress.js).

In an environment with 500 users and 50 tickets, we obtained the following test results.

<img src="/images/../../assets/images/2024-06-15-20-14-43.png" alt="Description" style="display:block; width:1200px; margin-left:auto; margin-right:auto;">

With random backoff applied, the average request time was about 760ms,

while with fixed backoff, the average request time was about 942ms.

A simple calculation of the improvement is as follows:

$$\text{Percentage Change} = \left( \frac{\text{New Value} - \text{Old Value}}{\text{Old Value}} \right) \times 100$$

$$
\text{Percentage Change} = \left( \frac{962 \, \text{ms} - 760 \, \text{ms}}{760 \, \text{ms}} \right) \times 100
$$

$$
\text{Percentage Change} = \left( \frac{202 \, \text{ms}}{760 \, \text{ms}} \right) \times 100 \approx 26.58\%
$$

We achieved about a 26% improvement.

## Appendix - Dynamic Retry Interval

Sometimes when retry logic is needed, you may need to set the retry range dynamically.

In our team’s case, when conducting the experiment “[How should we set the retry range?](https://www.notion.so/K6-72add37df17448c4bbf91585867deada?pvs=21)”, dynamic retry range configuration was necessary.

It would be convenient if you could easily write something like a template string:

`"#{T(java.util.concurrent.ThreadLocalRandom).current().nextInt(${min}, ${max})}"`

But unfortunately, this is not possible in SpeL.

To set the retry interval dynamically, you must use `RetryTemplate` instead of the annotation.

<img src="/images/../../assets/images/2024-06-15-20-18-16.png" alt="Description" style="display:block; width:800px; margin-left:auto; margin-right:auto;">

`RetryTemplate` conveniently provides an implementation called `UniformRandomBackoffPolicy()` that offers random intervals.

Using this implementation, you can set the minimum and maximum values, and build a `RetryTemplate` using the builder.

You can find the code [here](https://github.com/Tiketeer/Tiketeer-LockTest/blob/develop/src/main/java/com/tiketeer/Tiketeer/domain/purchase/usecase/CreatePurchaseOLockUseCase.java).

# Conclusion

In this document, we looked into **random backoff**, which can optimize performance in environments where multiple clients cause concurrency issues.

In environments where a queue is implemented, the number of users requesting simultaneously is limited, so the performance difference from the retry strategy can be minimal,

but in cases where many clients retry at the same fixed intervals, simply adding randomness to the retry interval can yield performance improvements.

Thank you.
