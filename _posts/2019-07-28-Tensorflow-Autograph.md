---
title:  "Benchmarking Tensorflow's autograph for arbitrary code"
date:   2019-07-28 08:07:40 -07:00
categories: Python Heterogeneous Tensorflow Quick 
layout: post
---

With the speed of light being unfortunately fixed, and the corollary that heterogeneous architectures with task specific data movement offer the path to higher FLOPs, moving existing CPU software to post-single-thread platforms is the price to pay for seeing post-2000's Moore's Law[^1].

TensorFlow ships a tool for automatically creating TF graphs from pretty general Python functions[^2]. This gives us a easy method for testing the overhead of "porting" a existing Python code to any TF target by converting it directly into a graph.

To test the compile and runtime performance, I grabbed a pure Python implementation of a cryptographic algorithm from [https://github.com/ajalt/python-sha1]. This was chosen despite SHA1 being easily implemented with hardware acceleration, as it should produce reasonable depth control flow graphs.

The compilation and runtime performance was benchmarked with the following loop with SHA1 depths from 1 to 1000.

{% highlight python %}
for iters in iter_counts:
    t0 = time.time()
    with tf.Graph().as_default():
        hfinal = tf.constant(h0)
        for _ in range(iters):
            hfinal = tf_sha1(hfinal)

        with tf.Session() as sess:
            result = sess.run(hfinal)

    dt = time.time()-t0
    print(iters, "iterations took", dt, "seconds")
{% endhighlight %}


![plot](/assets/tf_plot.png)

The SHA1 code was approximately 70 lines long. If you were to attempt to throw arbitrary Python into Autograph like this, you could reasonably expect it to take 21 seconds per kloc of Python.

Just do it properly.

References:

[^1]: [https://herbsutter.com/welcome-to-the-jungle/]
[^2]: [https://medium.com/tensorflow/autograph-converts-python-into-tensorflow-graphs-b2a871f87ec7]