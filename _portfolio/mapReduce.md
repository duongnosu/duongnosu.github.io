---
title: " Processing Big Data with `MapReduce` model  in `MongoDB` database. "
excerpt: "Implementing Merge Sort and Bucket Sort with `python` using `MapReduce` model  in `MongoDB` database. "
collection: portfolio
---

Implementing Merge Sort and Bucket Sort using `MapReduce` model  in `MongoDB` database.

## Introduction 
Big Data requires a different approach to distributed data storage that is designed for large-scale clusters.MapReduce is an important component when it comes to processing file. It is a programming model for processing large data sets with a parallel, distributed algorithm on a cluster. It is a framework for distributed computing that is designed to process large data sets in parallel by dividing the work into a set of independent tasks. 
There are 2 main functions of `MapReduce`, `_map_` function processes a key/value pair to generate a set of intermediate key/value pairs and `_reduce_` function merges all intermediate values associated with the same intermediate key. [^1]. 

[^1]: MapReduce(https://www.usenix.org/legacy/events/osdi04/tech/full_papers/dean/dean_html/)
