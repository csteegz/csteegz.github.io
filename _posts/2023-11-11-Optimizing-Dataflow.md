---
layout: post
title: Optimizing Apache Beam Jobs on Google Dataflow
date: 2022-11-11
---

Apache Beam is a powerful framework for processing big data pipelines in either batch or streaming mode, on a variety of runners. 
My team uses it to run terabyte to petabyte scale batch pipelines for feature generation.
Here I'll disucss ways to optimize Beam pipelines running on Google Cloud Dataflow. 
Somebody payed Google hundreds of thousands of dollars to learn these tricks; you get them for free!

This post is mostly focused on JVM Batch pipelines, as that's what my experience is with. 
I write pipelines and transforms in Scala, using both Beam and Scio. 
That said, it should translate pretty broadly - the advice is applicable to the standard Beam programming model, which means it should translate to other languages and streaming modes.  

# Cloud Dataflow's Cost Model

Google Cloud Dataflow charges you for two things, VM usage and Shuffle. When you perform a GBK, Dataflow calls out to the Google maintained "shuffle service", which efficiently distributes the work between machines. This means that you don't pay for the machine usage on this distribution, but do pay per-byte (with a 50% discount for the first TB) for the amount of data you shuffle. This means there's two ways to make your pipeline cheaper: 

1. Use less VM time
2. Make it shuffle less data

I'll talk about both of those in order now.
# Reducing VM Usage
Reducing the amount of VM usage is the first way to improve Beam performance.

## Machine Sizing
Remember, you pay for the VM's used to run your pipelines. Here's some tips on how to be effecient with those:
* Autoscaling is incredibly powerful, especially since different portions of Beam pipelines can have different compute profiles. Scale down to a few machines when you're IO bound, and up to many when CPU bound.
* If you have autoscaling on, you probably would prefer to use many small machines, vs few big machines. This might be different if you have different memory requirements, but the core/memory/cost ratio is contant.
* It's a good thing if your machines are nearly pegged on CPU usage. Don't pay for cores you aren't using.
* That being said, depending on your functions, you might need more memory. You can get heap dumps on OOM errors that you can look at in JFR to help spot memory leaks if this is an issue. 
* Dataflow allocates 1 worker thread per core by default - this means a standard GCP VM will give you 3.75 GB of memory for each worker. You can change this by either adjusting the worker count, or by using `highmem` machines which have double the memory per core.

## Google Cloud Profiler
* Google Cloud Profiler is super powerful and easy to enable for Dataflow.
It's imporant to remember, when processing large data, algorithmic complexity matters. The profiler will give you insight into which APIs don't scale to the data you're processing. For example, Scala's `List.Apply()` (which is list access) is linear, because the default Scala list is a Linked List. This adds up when processing trillions of input records. 
* In JVM world, if you see `SerializableCoder` (for Beam) in your profiles, you likely have something very inefficient going on. If you see `KryoCoder` (for Scio) you might have an opportunity for optimization. 

# Reducing Shuffle
Since you're billed per byte shuffled, and that also requires network traffic, reducing shuffle is one of the biggest things you can do to improve performance.

## Coders
Coders are responsible for converting from in memory objects to byte streams, and back again. 
When shuffle is being done, the byte streams are used for comparisons of keys for GBKs as well.
Coders are likely the single most important component of Beam performance. 
The coders in Beam's packages for the JVM are performant, with the exception of `SerilizableCoder`. 
This coder uses reflection - it's fine for your steps, but shouldn't be used for any data that you need to shuffle. 
Write a custom coder, or use a [schema](https://beam.apache.org/documentation/programming-guide/#schemas) with logical types; the Schema coder is performant.

If using Scio, take a look at it's documentation on [`KryoRegister`](https://spotify.github.io/scio/internals/Kryo.html) for low hanging fruit. 
You might be able to get improvements in performance above and beyond that with custom coders, but I haven't done any profiling with this.

## Combiners
`Combine.PerKey` is incredibly useful, and should be used wherever possible. 
It has an optimziation over GBK or Reduce, where it will combine on the worker nodes prior to shuffling, if possible. 
That being said, it's important to make sure that you have a good coder for your accumulator. Because the accumulator is what's shuffled, if it's large or the coder is inefficient you're going to have issues.   

## Joins
Shuffle in Beam is done with the `GroubByKey` primitive, which involves shuffling all the data that is going to be processed. If you have a different join profile, you might want to think about different ways to develop that. Scio has skew joins and other things that might be interesting, if you use Scala APIs. Broadcast joins can be done with Side Inputs, but be careful with fitting them into memory!   

# Other Services
Wile it's not as important for the performance of your pipelines, you're likely using GCP Dataflow because of it's integration with other services. These services also have impact on the performance. A few tips on some common GCS storage services. 

## Big Query
* Take a look at which APIs you're using to load your data into Big Query.
Loading data into BQ tends to be more efficient with Avro APIs then the comprable JSON APIs. In JVM land, that means avoiding the `TableRow` APIs for writing. That being said, you probably want to review the source on GitHub to follow what's the most efficient.
* You'll want to consider your Big Query slot usage. It might make sense to schedule your batch pipelines around when people are doing ad-hoc analysis to avoid requiring additional quota. 

## GCS
* Due to Beam "operator fusion", it can be easy to end up with overloaded workers or hot shards. You might want to shuffle before or after reads in-order to break fusion.  
* Dataflow has all it's code and piplines uploaded to your GCS accounts. If you're really desperate to manage costs, make sure to clean those up when you're done with them. 