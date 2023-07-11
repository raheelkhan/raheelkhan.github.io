---
title: Process CSV files with multiprocessing in Pandas
date: 2017-12-28
categories:
---

Pandas gives you the ability to [read large csv](https://pandas.pydata.org/pandas-docs/stable/generated/pandas.read_csv.html) in chunks using a iterator. This way you don’t have to load the full csv file into memory before you start processing.

My objective was to extract, transform and load (ETL) CSV files that is around 15GB.

Here is the code snippter that can be used to delegate jobs to multi cores to speed up a linear process.

```python
import pandas as pd
from multiprocessing import Pool

def process(df):
    """
    Worker function to process the dataframe.
    Do some heavy lifting
    """
    pass

# initialise the iterator object
iterator = pd.read_csv('file.csv', chunksize=200000, compression='gzip',
skipinitialspace=True, encoding='utf-8')
# depends on how many cores you want to utilise
max_processors = 4
# Reserve 4 cores for our script
pool = Pool(processes=max_processors)
f_list = []
for df in iterator:
    # pass the df to process functio in async manner
    f = pool.apply_async(process, [df])
    f_list.append(f)
    # Because of async nature it will keep delegating and at some point
    # the process will be killed by your OS because of OOM (out of memory)
    # so we have to make sure that it get resolved as soon as it it our max
    # proceses limit
    if len(f_list) == max_processors:
        for f in f_list:
            f.get()
            del f_list[:]
```
