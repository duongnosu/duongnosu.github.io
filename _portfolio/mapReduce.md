---
title: " Processing Big Data with `MapReduce` model  in `MongoDB` database. "
excerpt: "Implementing Merge Sort and Bucket Sort with `python` using `MapReduce` model  in `MongoDB` database. "
collection: portfolio
---

Implementing Merge Sort and Bucket Sort using `MapReduce` model  in `MongoDB` database.

## Introduction 
Big Data requires a different approach to distributed data storage that is designed for large-scale clusters. MapReduce is an important component when it comes to processing files in Big Data. It is a programming model for processing large data sets with a parallel, distributed algorithm on a cluster. It designed to process large data sets in parallel by dividing the work into a set of independent tasks. 
There are 2 main functions of `MapReduce`, `_map_` function processes a key/value pair to generate a set of intermediate key/value pairs and `_reduce_` function merges all intermediate values associated with the same intermediate key. [^1]. 

## Overview
In this post, I am going to implementing Merge Sort and Bucket Sort using `MapReduce` model  in `MongoDB` database. The main tasks are:
1. Calculate the number of movies released by each production company for each year in the dataset
2. Implement the Merge Sort algorithm and the Bucket Sort algorithm using two MapReduce programs, to sort results of Task 1.

#### Dataset:
Movies data set.
![Alt text](/images/MapReduce/dataset.png)








[^1]: MapReduce(https://www.usenix.org/legacy/events/osdi04/tech/full_papers/dean/dean_html/)

## 1. Calculate the number of movies released by each production company for each year in the dataset.
![MapReduce](/images/MapReduce/MapReduce.png)
1. **Data Retrieval**: Access the '`movie` -> `date`' path to obtain the release date. Extract release year.
2. **Company Identification**: Access the '`movie` -> `companies`' path to identify the production companies associated with the movie. Note that a movie may be jointly produced by multiple companies, we only consider  the top three production companies.
3. **Data Formatting**: Generate a series of pairs for each movie in the following format: `<year, company>`. This can be achieved by combining the movie's release year (Step 1) with the names of the production companies (Step 2).
4. **Data Storage**: Store `<year, company>` pairs of all movies into a text file. This text file will serve as the input data for the subsequent MapReduce program.
5. **MapReduce Implementation**:  using `mrjob` to calculate the frequency of each `<year, company>` pair. 

#### Code Implementation
```python
#extraction.py
import pymongo

if __name__ == "__main__":
    connection = pymongo.MongoClient('BD_ADDRESS')
    db = connection['COLLECTION_NAME']
    movies = db['DOCUMENT_NAME']
    #create output file
    f = open('year_and_company.txt', 'w')
    
    for movie in movies.find():
    #get the first 3 companies for each movie
        for i in range(min(len(movie['companies']), 3)):
         #get the year by extract the last 4 characters of the date,
         #concatenate with the company name and write to the output file
        year_comp = movie['date'][-4:] + "," + movie['companies'][i]['name']+'\n'
        f.write(year_comp)
    f.close()
```
```python
#mapreduce.py
from mrjob.job import MRJob
from mrjob.step import MRStep
import regex as re

class company_count(MRJob):
    def steps(self):
        return[MRStep(mapper= self.mapper1, reducer=self.reducer1)]
    # Add count 1 for each production company
    def mapper1(self, _,prod_comp):
        #yield prod_comp, max([counts])
        yield (prod_comp,1)
    #Count to the total count for each production company
    def reducer1(self, prod_comp, count):
        yield prod_comp, sum(count)

if __name__ == "__main__":
    company_count.run()
```
#### Sample Result
```text
"2008,Gavin McKinney Underwater Productions"	1
"2008,Gerber Pictures"	1
"2008,Global Entertainment Group"	1
"2008,Goldcrest Pictures"	2
...
"2008,Fox Atomic"	1
"2008,France 2 Cine9ma"	1
"2008,Futurikon"	1
"2008,Gary Sanchez Productions"	1
```
## 2. Implement the Merge Sort algorithm and the Bucket Sort algorithm using two MapReduce programs, to sort results.

### Merge Sort
![Merge](/images/MapReduce/Merge.png)
#### Code implementation
```python
#merge_sort.py
from mrjob.job import MRJob
from mrjob.step import MRStep
# to sort the <year, company> pairs in ascending order. 
# This sorting is based on the count of movies, and the sorted results
class count_pairs_mergesort(MRJob):
    def steps(self):
        return [
            MRStep(mapper=self.mapper, 
            reducer=self.reducer)
            ]
    #Mapper: Mapping the same dummy key "temp" to each  each <year, company>, count 
    def mapper(self, _, line):
        year_company, count = line.split('\t')
        count = int(count)
        yield "temp", (year_company[1:-1], count)
    #Merge sort
    def merge_sort(self, arr):
        #check if the array is empty or has only one element
        if len(arr) > 1:
            #divide the array into two halves Left and Right
            mid = len(arr) // 2
            left = arr[:mid]
            right = arr[mid:]
            #recursively call merge_sort() to further divide the Left and Right halves
            self.merge_sort(left)
            self.merge_sort(right)

            i = j = k = 0
            while i < len(left) and j < len(right):
                if left[i][1] < right[j][1]:  # Sort in ascending order by count
                    arr[k] = left[i]
                    i += 1
                else:
                    arr[k] = right[j]
                    j += 1
                k += 1

            while i < len(left):
                arr[k] = left[i]
                i += 1
                k += 1

            while j < len(right):
                arr[k] = right[j]
                j += 1
                k += 1
        return arr  # Return the sorted array
    #Reducer: Call merge sort and return <YEAR, COMPANY>, COUNT in the ascending order
    def reducer(self, _, records):
        sorted_records = self.merge_sort(list(records))
        for record in sorted_records:
            yield record[0], record[1]

if __name__ == '__main__':
    count_pairs_mergesort.run()
```
#### Sample Result
```text
"2008,Gary Sanchez Productions"	1
"2008,Futurikon"	1
"2008,France 2 Cine9ma"	1
"2008,Fox Atomic"	1
...
"2009,Relativity Media"	8
"1999,Touchstone Pictures"	9
"2000,Warner Bros."	9
"1998,Touchstone Pictures"	10
```
### Bucket Sort
![MapReduce](/images/MapReduce/Bucket.png)
#### Code implementation
```python
#bucket_sort.py
from mrjob.job import MRJob
from mrjob.step import MRStep

class count_pairs_bucketsort(MRJob):
    #Steps 
    def steps(self):
        return [
            MRStep(
                mapper=self.bucket_mapper,
                reducer=self.bucket_reducer),
            MRStep(
                reducer=self.final_step)
        ]   
    #Mapper: Assign each <year, company>, count pair to a bucket based on the count of movies.
    def bucket_mapper(self, _, line):
        year_company, count = line.split('\t')
        count = int(count) #convert type to int
        bucket_id = count // 3 #assign bucket id with each bucket size = 3
        yield bucket_id, (year_company[1:-1], count)
    #Reducer: Sort the <year, company>, count pairs in each bucket in descending order.
    def bucket_reducer(self, bucket_id, records):
        # Sort the records in each bucket in descending order
        sorted_records = sorted(records, key=lambda x: (-x[1], x[0]))  
        #create a dummy key "temp" 
        for record in sorted_records:
            yield "temp",(bucket_id, record)
    # Final step: Sort each bucket by its bucket_id in descending order and output the <year, company> pairs in each bucket in descending order.    
    def final_step(self, key, bucketid_records):
        for value in sorted(bucketid_records, key=lambda x:x[0], reverse=True):
            yield value[1]
if __name__ == '__main__':
    count_pairs_bucketsort.run()
```
#### Sample Result
```text
"1998,Touchstone Pictures"	10
"1999,Touchstone Pictures"	9
"2000,Warner Bros."	9
"2002,Warner Bros."	8
...
"2016,Warner Bros."	1
"2016,Weimaraner Republic Pictures"	1
"2016,Will Packer Productions"	1
"2016,Working Title Films"	1
```
