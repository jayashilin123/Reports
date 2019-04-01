

CREATE EXTERNAL TABLE ON JSONDUMP DATA

~~~ mysql
create external table if not exists electronics_jsondump_s3 (json_string string) 
location 's3://hivedatajkv/Amazon-review';
~~~~
~~~ mysql
select * from electronics_jsondump_s3 limit 3;
~~~
| electronics_jsondump_s3.json_string |
| --- |
|"{""reviewerID"": ""A2JXAZZI9PHK9Z"", ""asin"": ""0594451647"", ""reviewerName"": ""Billy G. Noland \""Bill Noland\"""", ""helpful"": [3, 3], ""reviewText"": ""I am using this with a Nook HD+. It works as described. The HD picture on my Samsung 52&#34; TV is excellent."", ""overall"": 5.0, ""summary"": ""HDMI Nook adapter cable"", ""unixReviewTime"": 1388707200, ""reviewTime"": ""01 3, 2014""}"|
|"{""reviewerID"": ""A2P5U7BDKKT7FW"", ""asin"": ""0594451647"", ""reviewerName"": ""Christian"", ""helpful"": [0, 0], ""reviewText"": ""The cable is very wobbly and sometimes disconnects itself.The price is completely unfair and only works with the Nook HD and HD+"", ""overall"": 2.0, ""summary"": ""Cheap proprietary scam"", ""unixReviewTime"": 1398556800, ""reviewTime"": ""04 27, 2014""}"|
|"{""reviewerID"": ""AAZ084UMH8VZ2"", ""asin"": ""0594451647"", ""reviewerName"": ""D. L. Brown \""A Knower Of Good Things\"""", ""helpful"": [0, 0], ""reviewText"": ""This adaptor is real easy to setup and use right out of the box. I had not problem with it at all, it is well worth the purchase. I recommend this adaptor very much for viewing your Nook videos on your HDTV. I just disagree with other reviews on the length of the adaptor, I found it to be fairly adequate as to how and where it is connected to my TV. For me it was just right not too long or too short, I was able to place my Nook right below the connection on the TV stand, it did not fall or anything else, it is fine. Use your own judgement, I'm too busy watching my movies :)"", ""overall"": 5.0, ""summary"": ""A Perfdect Nook HD+ hook up"", ""unixReviewTime"": 1399161600, ""reviewTime"": ""05 4, 2014""}"|

___
CREATE SCHEMA TABLE ON S3 DATA

~~~ mysql
drop table electronics_columns_s3;
~~~

~~~ mysql
create external table if not exists electronics_columns_s3(
reviewerid string, 
asin string, 
reviewername string, 
helpful array<int>, 
reviewtext string, 
overall double, 
summary string, 
unixreviewtime bigint) 
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe' 
with serdeproperties  ('paths' = '')
location 's3://hivedatajkv/Amazon-review';
~~~

~~~ mysql
select reviewerid, reviewername, asin, unixreviewtime from electronics_columns_s3 limit 3;
~~~

| reviewerid | reviewername | asin | unixreviewtime |
| --- | --- | --- | --- |
| AO94DHGC771SJ | amazdnu | 0528881469 | 1370131200 |
| AMO214LNFCEI4 | Amazon Customer | 0528881469 | 1290643200 |
| A3N7T0DY83Y4IG | C. A. Freeman | 0528881469 | 1283990400 |
___

NUMBER OF REVIEWERS

~~~ mysql
select count(*) as review_count, count(distinct reviewerid) as reviewer_count,
count(distinct asin) as product_count, min(unixreviewtime) as min_time, 
max(unixreviewtime) as max_time
from electronics_columns_s3;
~~~
| review_count | reviewer_count | product_count | min_time | max_time |
| --- | --- | --- | --- | --- |
| 1689188 | 192403 | 63001 | 929232000 | 1406073600 |
__

AVERAGE REVIEWS PER REVIEWER

~~~ mysql
select avg(review_count) 
from (
  select reviewerid, count(*) as review_count
  from electronics_columns_s3 
  group by reviewerid
)a;
~~~
| _c0 |
| --- |
| 8.779426516218562 |
___

REVIEWS BY DATE
~~~ mysql
select year(from_unixtime(unixreviewtime)) as yr, count(*) as review_count
from electronics_columns_s3
group by year(from_unixtime(unixreviewtime)):
~~~
| yr | review_count |
| --- | --- |
| 2010 | 103797 |
| 2005 | 9638 |
| 2006 | 15447 |
| 2008 | 49872 |
| 2003 | 3547 |
| 2007 | 35976 |
| 2009 | 70666 |
| 2012 | 282942 |
| 2002 | 2315 |
| 2014 | 341188 |
| 2000 | 817 |
| 1999 | 72 |
| 2001 | 1609 |
| 2004 | 5159 |
| 2011 | 173395 |
| 2013 | 592748 |

![Year vs Review count](https://github.com/jayashilin123/Reports/blob/master/pic/plot01.jpg)
___

PARTITION THE DATA
~~~ mysql
drop table electronics_columns_s3_partitioned_year_month;
~~~
~~~ mysql
create external table if not exists electronics_columns_s3_partitioned_year_month
(reviewerid string, asin string, reviewername string, helpful array<int>, reviewtext string,
overall double, summary string, unixreviewtime bigint) partitioned by (yr int, mnth int)
location 's3://hivedatajkv/amazonelectronics_partition_year_month';


SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;

insert overwrite table electronics_columns_s3_partitioned_year_month partition(yr, mnth)
select reviewerid, asin, reviewername, helpful, reviewtext, overall, summary, unixreviewtime, 
year(from_unixtime(unixreviewtime)) as yr, month(from_unixtime(unixreviewtime)) as mnth
from electronics_columns_s3:
~~~
___

NON PARTITIONED MONTH QUERY RATINGS DISTRIBUTION FOR A MONTH
~~~ mysql
select overall, count(*) as review_count
from electronics_columns_s3
where year(from_unixtime(unixreviewtime)) = 2004
and month(from_unixtime(unixreviewtime)) = 1
group by overall;
~~~

PARTITIONED MONTH QUERY RATINGS DISTRIBUTION FOR A MONTH
~~~ mysql
select overall, count(*) as review_count
from electronics_columns_s3_partitioned_year_month
where yr = 2004 and mnth = 1
group by overall:
~~~
| overall | review_count |
| --- | --- |
| 1.0 | 51 |
| 2.0 | 29 |
| 3.0 | 36 |
| 4.0 | 71 |
| 5.0 | 180 |

![overall rating vs review count](https://github.com/jayashilin123/Reports/blob/master/pic/plot02.jpg)
___

CLUSTERING/BUCKETING
~~~ mysql
drop table electronics_columns_s3_partitioned_year_month_clustered;
~~~
~~~ mysql
create external table if not exists electronics_columns_s3_partitioned_year_month_clustered
(reviewerid string, asin string, reviewername string, helpful array<int>, reviewtext string,
overall double, summary string, unixreviewtime bigint) partitioned by (yr int, mnth int)
clustered by (reviewerid) into 4 buckets
location 's3://hivedatajkv/electronics_columns_s3_partitioned_year_month_clustered';

set hive.exec.max.dynamic.partitions=1000;
set hive.exec.max.dynamic.partitions.pernode=1000;

insert overwrite table electronics_columns_s3_partitioned_year_month_clustered partition(yr, mnth)
select reviewerid, asin, reviewername, helpful, reviewtext,
overall, summary, unixreviewtime, yr, mnth
from electronics_columns_s3_partitioned_year_month;
~~~
___

PRODUCT POPULARITY VS RATING
~~~ mysql
select popularity, count(*) as products
from(
    select `asin`, count(*) as popularity
    from electronics_columns_s3_partitioned_year_month
    where yr = 2012 
    group by `asin`
)a
group by popularity;
~~~
Reproducing few results for ref

| popularity | products
| --- | --- |
| 1	| 10169 |
| 2	| 7089 |
| 3	| 4920 |
| 4	| 3358 |
| 5 | 2346 |
| 6	| 1694 |
| 7	| 1180 |

![product vs popularity](https://github.com/jayashilin123/Reports/blob/master/pic/plot-3.jpg)
___

CORRELATION BETWEEN REVEWTEXT AND RATING
~~~ mysql
select size(split(reviewtext, ' ')) as words, avg(overall)
from electronics_columns_s3_partitioned_year_month
where yr = 2013
group by size(split(reviewtext, ' '));
~~~
Reproducing few results for ref

| words | _c1 |
| --- | --- |
| 4 | 5.0 |
| 5 | 5.0 |
| 9 | 3.7142857142857144 |
| 10 | 3.0 |
| 12 | 4.4 |
| 14 | 4.7272727272727275 |
| 16 | 4.651515151515151 |

___
CONVERT FILE FORMAT TO ORC
~~~ mysql
drop table electronics_columns_s3_partitioned_year_month_orc;
create external table if not exists electronics_columns_s3_partitioned_year_month_orc
(reviewerid string, asin string, reviewername string, helpful array<int>, reviewtext string,
overall double, summary string, unixreviewtime bigint) partitioned by (yr int, mnth int)
stored as orc location 's3://hivedatajkv/amazonelectronics_partition_year_month_orc'
tblproperties ("orc.compress"="SNAPPY");

insert overwrite table electronics_columns_s3_partitioned_year_month_orc partition(yr , mnth)
select * from electronics_columns_s3_partitioned_year_month;
~~~
___

FINDING THE LENGTH OF THE USER REVIEWS 
~~~ mysql
select avg( size(split(reviewtext, ' ')) ) as avg_length, variance( size(split(reviewtext, ' ')) ) as varinc
from electronics_columns_s3_partitioned_year_month_orc
where yr = 2013 and mnth = 1;
~~~
| avg_length | varinc |
| --- | --- |
| 89.96355594	| 15897.49022 |

~~~ mysql
select size(split(reviewtext, ' ')) as words, avg(overall) as avg_rating
from electronics_columns_s3_partitioned_year_month_orc
where yr = 2013 and mnth = 1
group by size(split(reviewtext, ' '));
~~~
| words	| _c1 |
| --- | --- |
| 4	| 5 |
| 5	| 5 |
| 9	| 3.714285714 |
| 10	| 3 |
| 12	| 4.414	|
| 14 | 4.727272727 |
| 16 | 4.651515152 |

![word vs avg_rating](https://github.com/jayashilin123/Reports/blob/master/pic/plot04.jpg)
___

TEXT ANALYSIS USING (N-GRAM)
~~~ mysql
select reviewtext, sentences(lower(reviewtext)) 
from electronics_columns_s3_partitioned_year_month_orc
where yr = 2013 and mnth = 1
limit 1;
~~~

| reviewtext |_c1 |
| --- | --- |
|  Great mouse. Good battery life, good connection, good tracking on my couch. Feels good in my hand, easy to use.	| [["great","mouse","good","battery","life","good","connection","good","tracking","on","my","couch","feels","good","in","my","hand","easy","to","use"]] |

~~~ mysql
SELECT explode( ngrams( sentences( lower(reviewtext) ), 2, 6))
FROM electronics_columns_s3_partitioned_year_month_orc 
where yr = 2013 and mnth = 1;
~~~
| col |
| --- |
| {"ngram":["of","the"],"estfrequency":20747.0} |
| {"ngram":["i","have"],"estfrequency":15992.0} |
| {"ngram":["on","the"],"estfrequency":15576.0} |
| {"ngram":["it","is"],"estfrequency":14290.0} |
| {"ngram":["in","the"],"estfrequency":13852.0} |
| {"ngram":["for","the"],"estfrequency":12946.0} |
___

TOP 10 PRODUCTS BY MONTH
~~~ mysql
select helpful[0] as pstv, helpful[1] as total
from electronics_columns_s3_partitioned_year_month_orc
where yr = 2013  and
helpful[1] != 0;
~~~
| pstv |	total |
| --- | --- |
| 4	| 5 |
| 1	| 1 |
| 2	| 4 |
| 2	| 2 |
| 1	| 2 |
| 5	| 5 |
| 3	| 3 |
| 0	| 1 |

