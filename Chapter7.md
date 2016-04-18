### Testing And Validation 
Testing query performance was carried out on both random data and on the real data from the current app-usage system for SQL, MongoDB and InfluxDB. While generating test-data it is important to make it as close to the real data as possible in terms of scale and the dynamic nature of the data. This is why the tests on the random data and the real data showed completely different results. As far as possible, testing is better done on the real data . If that's not possible or convenient, the test-data generated must be as close to the real data as possible to get the right idea about each database's performance.
#### Random Data 
About 50 million records was generated using Python's random module. The pseudo-random generator in python follows a near perfect uniform distribution, so the distribution of data was even. The data was logged into SQL and MongoDB. The results of the evaluation in [here]() show that with a fixed schema and millions of records, MongoDB performed poorly.Performance was better for fewer records, obviously because now the working set fits in RAM and there was less disk I/O. So, in spite of MongoDB's poor performance we decided to go with it because of its flexible data model. Since our use case could be considered a time-series model(basically,usage analysis is just grouping/counting and observing trends over time), InfluxDB seemed to be worth considering. With a similar dataset of 50m row, it outperformed MongoDB and was comparable to SQL, as shown [here]().

#### Real Data 
Testing on real data was done to make certain if InfluxDB performed as well. It was expected to, since it handled 50m records well and our scale was much smaller than that. But due to the series cardinality of the real data, InfluxDB started to consume more and more memory during queries and performance was poor for queries that involved merging a lot of series. The random test data contained about 128 series, the real data about 605,000. This was far more than the recommended 100,000 InfluxDB could handle. MongoDB, when the scale was reduced from 50million to something more realistic, was very fast because there was minimal disk I/O involved. SQL's performance suffered because the current app-usage system was normalized and atleast 6-7 joins were needed to extract meaningful data.


Python Scripts to load random data and test :
1) mongo_writer.py
2) sql_writer.py
3) mongo_tests.py
4) sql_tests.py
5) influx_writer.py
6) influx_tests.py
7) log_generator.py

Python scripts to load real data 
1) get_usage_data.py
2) mongo_usage_data.py
3) ??

