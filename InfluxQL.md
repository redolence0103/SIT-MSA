# InfluxQL

InfluxQL URL : https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/data_exploration/index

### SELECT .. FROM ..


```sql
$ influx -username=admin -password=wjdgid0103 -precision rfc3339 -database SIT_PV
Connected to http://localhost:8086 version 1.8.10
InfluxDB shell version: 1.8.10
> use atid1024
# Quoting
[A-z,0-9,_] 이외의 문자가 있거나 숫자로 시작하거나 InfluxQL 키워드 인 경우 식별자 는 큰 따옴표 로 묶어야합니다 . 항상 필요한 것은 아니지만 식별자를 큰 따옴표로 묶는 것이 좋습니다.
# 단일 측정에서 모든 field 및 tag 선택 (LIMIT 5 : 5개 raw로 제한)
> SELECT * FROM "AI028" limit 5
name: AI028
time                cdevice dtype endpoint pdevice sep   value
----                ------- ----- -------- ------- ---   -----
1639958400000000000 WTS0001 E     ATMPS    WTS0000 '001' -4.7
1639958400000000000 WTS0001 E     HIRS     WTS0000 '001' 0
1639958400000000000 WTS0001 E     MTMPS    WTS0000 '001' -6.9
1639958400000000000 WTS0001 E     SIRS     WTS0000 '001' 0
1639958460000000000 ACB0001 E     FRQ      TRN0001 '001' 59.9961

# 단일 측정에서 특정 tag 및 field 선택 (field 값은 하나이상 포함되어야 한다.)
> SELECT "pdevice","cdevice" FROM "AI028" limit 5
> SELECT "endpoint","value" FROM "AI028" limit 5
name: AI028
time                endpoint value
----                -------- -----
1639958400000000000 ATMPS    -4.7
1639958400000000000 HIRS     0
1639958400000000000 MTMPS    -6.9
1639958400000000000 SIRS     0
1639958460000000000 AOHP     0

# 단일 측정에서 모든 field 선택
> SELECT *::field FROM "AI028" LIMIT 5
name: AI028
time                value
----                -----
1639958400000000000 -4.7
1639958400000000000 0
1639958400000000000 -6.9
1639958400000000000 0
1639958460000000000 -6.9

# 측정에서 특정 field를 선택하고 기본 산술을 수행 
(influxDB 제공 수학연산자: https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/math_operators/index)
> SELECT ("value" * 10) FROM "AI028" LIMIT 5
name: AI028
time                value
----                -----
1639958400000000000 -47
1639958400000000000 0
1639958400000000000 -69
1639958400000000000 0
1639958460000000000 -69
# 둘 이상의 측정에서 모든 데이터 선택
> select * from "AI028", "AI030" limit 5
name: AI028
time                cdevice dtype endpoint pdevice sep   value
----                ------- ----- -------- ------- ---   -----
1639958400000000000 WTS0001 E     ATMPS    WTS0000 '001' -4.7
1639958400000000000 WTS0001 E     HIRS     WTS0000 '001' 0
1639958400000000000 WTS0001 E     MTMPS    WTS0000 '001' -6.9
1639958400000000000 WTS0001 E     SIRS     WTS0000 '001' 0
1639958460000000000 ACB0001 E     FRQ      TRN0001 '001' 59.9961

name: AI030
time                cdevice dtype endpoint pdevice sep   value
----                ------- ----- -------- ------- ---   -----
1639958400000000000 WTS0001 E     ATMPS    WTS0000 '001' -4.7
1639958400000000000 WTS0001 E     HIRS     WTS0000 '001' 0
1639958400000000000 WTS0001 E     MTMPS    WTS0000 '001' -6.9
1639958400000000000 WTS0001 E     SIRS     WTS0000 '001' 0
1639958460000000000 ACB0001 E     FRQ      TRN0001 '001' 59.9961
> 
```



### GROUP BY 시간 간격 문제 (offset)


```bash
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE time >= '2019-08-18T00:00:00Z' AND time <= '2019-08-18T00:30:00Z' GROUP BY time(12m),"location"
name: h2o_feet
tags: location=coyote_creek
time                 count
----                 -----
2019-08-18T00:00:00Z 2
2019-08-18T00:12:00Z 2
2019-08-18T00:24:00Z 2

name: h2o_feet
tags: location=santa_monica
time                 count
----                 -----
2019-08-18T00:00:00Z 2
2019-08-18T00:12:00Z 2
2019-08-18T00:24:00Z 2
> SELECT "water_level" FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2019-08-18T00:00:00Z' AND time <= '2019-08-18T00:18:00Z'
name: h2o_feet
time                 water_level
----                 -----------
2019-08-18T00:00:00Z 8.504
2019-08-18T00:06:00Z 8.419
2019-08-18T00:12:00Z 8.32
2019-08-18T00:18:00Z 8.225
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2019-08-18T00:06:00Z' AND time < '2019-08-18T00:18:00Z' GROUP BY time(12m)
name: h2o_feet
time                 count
----                 -----
2019-08-18T00:00:00Z 1
2019-08-18T00:12:00Z 1
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2019-08-18T00:06:00Z' AND time < '2019-08-18T00:18:00Z' GROUP BY time(12m,6m)
name: h2o_feet
time                 count
----                 -----
2019-08-18T00:06:00Z 2
```



### GROUP BY 시간 간격  fill()

`fill()` 은 데이터가없는 시간 간격에 대해보고 된 값을 변경합니다.

- 임의의 숫자 데이터없이 시간 간격에 대해 주어진 숫자 값을보고합니다.

- linear` 데이터가없는 시간 간격에 대한 [선형 보간](https://en.wikipedia.org/wiki/Linear_interpolation) 결과를보고합니다 .

- `none` 타임 스탬프 및 데이터가없는 시간 간격 값을보고하지 않습니다.

- `null` 데이터가없는 시간 간격에 대해 null을 보고하지만 타임 스탬프를 반환합니다. 이것은 기본 동작과 동일합니다.

`previous` 데이터가없는 시간 간격에 대해 이전 시간 간격의 값을보고합니다.


```bash

```



### INTO

InfluxDB에서 데이터베이스를 직접 이름을 바꿀 수 없으므로 `INTO` 절의 일반적인 용도 는 한 데이터베이스에서 다른 데이터베이스로 데이터를 이동하는 것입니다. 아래 쿼리는 `NOAA_water_database` 및 `autogen` 보존 정책 의 모든 데이터를 `copy_NOAA_water_database` 데이터베이스 및 `autogen` 보존 정책에 씁니다 .


```sql
SELECT * INTO "copy_NOAA_water_database"."autogen".:MEASUREMENT FROM "NOAA_water_database"."autogen"./.*/ GROUP BY *
name: result
time written
---- -------
0    76290
```

- 많은 양의 데이터를 이동할 때는 다른 측정에 대해 `INTO` 쿼리를 순차적으로 실행 하고 [`WHERE` 절의](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/data_exploration/index#time-syntax) 시간 경계를 사용 하는 것이 좋습니다 . 이렇게하면 시스템 메모리가 부족해지지 않습니다. 아래 코드 블록은 해당 쿼리에 대한 샘플 구문을 제공합니다.

  ```
  SELECT * 
  INTO <destination_database>.<retention_policy_name>.<measurement_name> 
  FROM <source_database>.<retention_policy_name>.<measurement_name>
  WHERE time > now() - 100w and time < now() - 90w GROUP BY *
  
  SELECT * 
  INTO <destination_database>.<retention_policy_name>.<measurement_name> 
  FROM <source_database>.<retention_policy_name>.<measurement_name>} 
  WHERE time > now() - 90w  and time < now() - 80w GROUP BY *
  
  SELECT * 
  INTO <destination_database>.<retention_policy_name>.<measurement_name> 
  FROM <source_database>.<retention_policy_name>.<measurement_name>
  WHERE time > now() - 80w  and time < now() - 70w GROUP BY *
  ```

  

- #### 측정 결과에 쿼리 결과 쓰기

  ```
  > SELECT "water_level" INTO "h2o_feet_copy_1" FROM "h2o_feet" WHERE "location" = 'coyote_creek'
  
  name: result
  ------------
  time                   written
  1970-01-01T00:00:00Z   7604
  
  > SELECT * FROM "h2o_feet_copy_1"
  
  name: h2o_feet_copy_1
  ---------------------
  time                   water_level
  2015-08-18T00:00:00Z   8.12
  [...]
  2015-09-18T16:48:00Z   4
  ```



- 집계 결과를 측정에 기록 (다운 샘플링)

  ```
  > SELECT MEAN("water_level") INTO "all_my_averages" FROM "h2o_feet" WHERE "location" = 'coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)
  
  name: result
  ------------
  time                   written
  1970-01-01T00:00:00Z   3
  
  > SELECT * FROM "all_my_averages"
  
  name: all_my_averages
  ---------------------
  time                   mean
  2015-08-18T00:00:00Z   8.0625
  2015-08-18T00:12:00Z   7.8245
  2015-08-18T00:24:00Z   7.5675
  ```



- 하나 이상의 측정에 대한 집계 결과를 다른 데이터베이스에 기록 (역 참조를 통한 다운 샘플링)

  ```
  > SELECT MEAN(*) INTO "where_else"."autogen".:MEASUREMENT FROM /.*/ WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:06:00Z' GROUP BY time(12m)
  
  name: result
  time                   written
  ----                   -------
  1970-01-01T00:00:00Z   5
  
  > SELECT * FROM "where_else"."autogen"./.*/
  
  name: average_temperature
  time                   mean_degrees   mean_index   mean_pH   mean_water_level
  ----                   ------------   ----------   -------   ----------------
  2015-08-18T00:00:00Z   78.5
  
  name: h2o_feet
  time                   mean_degrees   mean_index   mean_pH   mean_water_level
  ----                   ------------   ----------   -------   ----------------
  2015-08-18T00:00:00Z  
  ```

- into 절의 일반적인 문제

  #### 문제 1 : 데이터 누락

  `INTO` 쿼리 에 [`SELECT` ](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/data_exploration/index#the-basic-select-statement)절 에 [태그 키](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#tag-key) 가 포함되어 있으면 쿼리는 현재 측정의 [태그](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#tag) 를 대상 측정의 [필드](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#field) 로 변환 합니다 . 이로 인해 InfluxDB 는 이전에 [태그 값으로](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#tag-value) 구분되었던 [포인트](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#point) 를 덮어 쓸 수 있습니다 . [`TOP()` ](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/functions/index#top)또는 [`BOTTOM()` ](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/functions/index#bottom)함수 를 사용하는 쿼리에는이 동작이 적용되지 않습니다 . [자주 묻는 질문](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/troubleshooting/frequently-asked-questions/index#why-are-my-into-queries-missing-data) 문서는 세부에서 해당 동작을 설명합니다.

  대상 측정에서 태그로 전류 측정에 태그를 유지하려면 [`GROUP BY` 관련 태그 키](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/data_exploration/index#group-by-tags) 또는 `GROUP BY *` 에서 `INTO` 의 쿼리.

  #### 문제 2 : `INTO` 절을 사용하여 쿼리 자동화

  이 문서 의 `INTO` 절 섹션은 `INTO` 절을 사용하여 쿼리를 수동으로 구현하는 방법을 보여줍니다 . 실시간 데이터에 대한 `INTO` 절 쿼리 를 자동화하는 방법 은 [연속 쿼리](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/index#continuous_queries) 설명서를 참조하십시오 . 연속 쿼리 [는 다른 용도](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/continuous_queries/index#continuous-query-use-cases) 중에서 다운 샘플링 프로세스를 자동화합니다.

### 시간별 주문 

기본적으로 InfluxDB는 오름차순으로 결과를 반환합니다. 반환 된 첫 번째 [포인트](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#point) 에는 가장 오래된 [타임 스탬프가](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#timestamp) 있고 반환 된 마지막 포인트에는 가장 최근의 타임 스탬프가 있습니다. `ORDER BY time DESC` 는 순서를 반대로하여 InfluxDB가 가장 최근 타임 스탬프가있는 포인트를 먼저 반환합니다.

- 최신 포인트를 먼저 반환

  ```
  > SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' ORDER BY time DESC
  
  name: h2o_feet
  time                   water_level
  ----                   -----------
  2015-09-18T21:42:00Z   4.938
  2015-09-18T21:36:00Z   5.066
  [...]
  2015-08-18T00:06:00Z   2.116
  2015-08-18T00:00:00Z   2.064
  ```

- 최신 포인트를 먼저 리턴하고 GROUP BY time() 포함

  기본적으로 InfluxDB는 오름차순으로 결과를 반환합니다. 반환 된 첫 번째 [포인트](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#point) 에는 가장 오래된 [타임 스탬프가](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#timestamp) 있고 반환 된 마지막 포인트에는 가장 최근의 타임 스탬프가 있습니다. `ORDER BY time DESC` 는 순서를 반대로하여 InfluxDB가 가장 최근 타임 스탬프가있는 포인트를 먼저 반환합니다.

  ```
  > SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:42:00Z' GROUP BY time(12m) ORDER BY time DESC
  
  name: h2o_feet
  time                   mean
  ----                   ----
  2015-08-18T00:36:00Z   4.6825
  2015-08-18T00:24:00Z   4.80675
  2015-08-18T00:12:00Z   4.950749999999999
  2015-08-18T00:00:00Z   5.07625
  ```

  

### LIMIT 및 SLIMIT 

`LIMIT` 및 `SLIMIT` 는 쿼리 당 반환 되는 [포인트](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#point) 수와 [시리즈](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#series) 수를 제한합니다 .

```
> SELECT "water_level","location" FROM "h2o_feet" LIMIT 3

name: h2o_feet
time                   water_level   location
----                   -----------   --------
2015-08-18T00:00:00Z   8.12          coyote_creek
2015-08-18T00:00:00Z   2.064         santa_monica
2015-08-18T00:06:00Z   8.005         coyote_creek
```



- 리턴되는 수를 제한하고 GROUP BY 절을 포함

  ```
  > SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:42:00Z' GROUP BY *,time(12m) LIMIT 2
  
  name: h2o_feet
  tags: location=coyote_creek
  time                   mean
  ----                   ----
  2015-08-18T00:00:00Z   8.0625
  2015-08-18T00:12:00Z   7.8245
  
  name: h2o_feet
  tags: location=santa_monica
  time                   mean
  ----                   ----
  2015-08-18T00:00:00Z   2.09
  2015-08-18T00:12:00Z   2.077
  ```

  

-  `SLIMIT` 의 절

  `SLIMIT <N>` 은 지정된 [측정](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#measurement) 에서 <N> [시리즈의](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#series) 모든 [지점](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#point) 을 반환 합니다

```
# 반환된 수 제한
> SELECT "water_level" FROM "h2o_feet" GROUP BY * SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   water_level
----                   -----
2015-08-18T00:00:00Z   8.12
2015-08-18T00:06:00Z   8.005
2015-08-18T00:12:00Z   7.887
[...]
2015-09-18T16:12:00Z   3.402
2015-09-18T16:18:00Z   3.314
2015-09-18T16:24:00Z   3.235

# 리턴 된 시리즈 수를 제한하고 GROUP BY time() 절 포함
> SELECT "water_level" FROM "h2o_feet" GROUP BY * SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   water_level
----                   -----
2015-08-18T00:00:00Z   8.12
2015-08-18T00:06:00Z   8.005
2015-08-18T00:12:00Z   7.887
[...]
2015-09-18T16:12:00Z   3.402
2015-09-18T16:18:00Z   3.314
2015-09-18T16:24:00Z   3.235
```



- 제한 및 제한

쿼리는 [측정 ](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#measurement)`h2o_feet` 와 관련된 [시리즈](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#series) 중 하나에서 세 개의 가장 오래된 [포인트](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/concepts/glossary/index#point) (타임 스탬프로 결정)를 반환합니다

```
# 반환된 포인트 및 시리즈 수 제한
> SELECT "water_level" FROM "h2o_feet" GROUP BY * LIMIT 3 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   water_level
----                   -----------
2015-08-18T00:00:00Z   8.12
2015-08-18T00:06:00Z   8.005
2015-08-18T00:12:00Z   7.887
```

쿼리는 InfluxQL [함수](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/functions/index) 와 [GROUP BY 절의](https://runebook.dev/ko/docs/influxdata/influxdb/v1.3/query_language/data_exploration/index#group-by-time-intervals) 시간 간격을 사용하여 쿼리 시간 범위에서 각 12 분 간격 의 평균 `water_level` 을 계산합니다 . `LIMIT 2` 는 두 개의 가장 오래된 12 분 평균 (타임 스탬프에 의해 `SLIMIT 1` )을 요청 하고 SLIMIT 1 은 `h2o_feet` 측정 과 관련된 단일 시리즈를 요청합니다 .

```
> SELECT MEAN("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:42:00Z' GROUP BY *,time(12m) LIMIT 2 SLIMIT 1

name: h2o_feet
tags: location=coyote_creek
time                   mean
----                   ----
2015-08-18T00:00:00Z   8.0625
2015-08-18T00:12:00Z   7.8245
```



- 시간 산술 연산

  이 쿼리는 2015 년 9 월 18 일 21:24:00에 최소 6 분 후에 발생하는 타임 스탬프가있는 데이터를 반환합니다. `+` 와 `6m` 사이의 공백 이 필요합니다.

  ```
  > SELECT "water_level" FROM "h2o_feet" WHERE time > '2015-09-18T21:24:00Z' + 6m
  
  name: h2o_feet
  time                   water_level
  ----                   -----------
  2015-09-18T21:36:00Z   5.066
  2015-09-18T21:42:00Z   4.938
  ```

- 상대시간

  `u` 또는 `µ` 마이크로 초 `ms` 밀리 초 `s` 초 `m` 분 `h` 시간 `d` 일 `w` 주

  ```
  > SELECT "water_level" FROM "h2o_feet" WHERE time > '2015-09-18T21:24:00Z' + 6m
  
  name: h2o_feet
  time                   water_level
  ----                   -----------
  2015-09-18T21:36:00Z   5.066
  2015-09-18T21:42:00Z   4.938
  ```

- 시간 구문의 일반적인 문제

  현재 InfluxDB는 `WHERE` 절 에서 절대 시간으로 `OR` 사용을 지원하지 않습니다 .
  
- 집계 결과를 측정에 기록 (다운 샘플링)

  ```sql
  > select difference(max("value")) as "TAE" into "pvef" from "AI028" where time >= '2021-12-20T00:00:00Z' and time <= '2021-12-23T23:59:00Z' and "endpoint" = 'TAE' and "cdevice" = 'VCB0001'  group by time(30m)
  
  > select mean("value")+0.001 as "SIRS" into "pvef" from "AI028" where time >= '2021-12-20T00:00:00Z' and time <= '2021-12-23T23:59:00Z' and "endpoint" = 'SIRS' and "cdevice" = 'WTS0001'  group by time(30m)
  
  > select 100*"TAE"/"SIRS"  from "pvef" 
  
  ```
  
  ```sql
  > select difference(max("value")) as "TAE" into statistics..AI028_1H from "AI028" where time >= '2021-12-20T00:00:00Z' and time <= '2021-12-23T23:59:00Z' and "endpoint" = 'TAE' and "cdevice" = 'VCB0001'  group by time(60m)
  
  > select mean("value")+0.001 as "SIRS" into statistics..AI028_1H from "AI028" where time >= '2021-12-20T00:00:00Z' and time <= '2021-12-23T23:59:00Z' and "endpoint" = 'SIRS' and "cdevice" = 'WTS0001'  group by time(60m)  
  ```

  

- 현재 값과 지난값 비교하여 표시

  ```sql
  > select difference(mean("value")) from "AI028" where time >= '2021-12-20T08:00:00Z' and time <= '2021-12-20T18:59:00Z' and "endpoint" = 'TAE'  group by time(15m) limit 10
  name: AI028
  time                 difference
  ----                 ----------
  2021-12-20T08:00:00Z 3.8062400000635535
  2021-12-20T08:15:00Z 4.418760000029579
  2021-12-20T08:30:00Z 10.184379999758676
  2021-12-20T08:45:00Z 13.421880000038072
  2021-12-20T09:00:00Z 17.240610000211746
  2021-12-20T09:15:00Z 17.64686999982223
  2021-12-20T09:30:00Z 20.350010000169277
  2021-12-20T09:45:00Z 29.962489999830723
  2021-12-20T10:00:00Z 57.36563999997452
  2021-12-20T10:15:00Z 65.60312000010163
  ```




- telegraf(influxDB kafa listener )

  https://archive.docs.influxdata.com/telegraf/v1.8/introduction/installation/

```bash
wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
  source /etc/lsb-release
  echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
  
  sudo apt-get update && sudo apt-get install telegraf
  sudo systemctl start telegraf
  
  /etc/telegraf/telegraf.conf
  
  # # Read metrics from Kafka topics
 [[inputs.kafka_consumer]]
#   ## Kafka brokers.
   brokers = ["localhost:9092"]
#
#   ## Topics to consume.
   topics = ["telegraf"]
   
   data_format = "influx"
```
                                                                                                     
## MVP data 전처리
```bash
cat -v sample-pv.txt |more
influx -import -path=sample-pv.txt -precision=s -database=atid1024 -username=admin -password=wjdgid0103
                                                                                                     
select 0.001+sum("value")/60 as "SIRS" into "PR" from "AI028" where time >= '2021-12-20T00:00:00Z' and time <= '2021-12-23T23:59:00Z' and "endpoint" = 'SIRS'  group by time(60m)
select difference(max("value")) as "TAE" into "PR" from "AI028" where time >= '2021-12-20T00:00:00Z' and time <= '2021-12-23T23:59:00Z' and "endpoint" = 'TAE'  group by time(60m)
select 100*((TAE/498.96)/((SIRS/1000))) as PR into "PR" from "PR" group by 
                                                                                                                   
# Generator(file에서 분단위로 line을 읽어서 Parser 서비스에 전송)

  CREATE CONTINUOUS QUERY "PR_1H" ON "atid1024" BEGIN  SELECT difference(max("value")) as "TAE" into "a_year"."PR" from "AI028" GROUP BY time(60m) END
  CREATE CONTINUOUS QUERY "PR_SIRS_1H" ON "atid1024" BEGIN select 0.001+sum("value")/60 as "SIRS" into "a_year"."PR" from "AI028" GROUP BY time(60m) END
  CREATE CONTINUOUS QUERY "PR_PR_1H" ON "atid1024" BEGIN select 100*((TAE/498.96)/((SIRS/1000))) as PR into a_year"."PR" from "PR" group by * END                                                                                                                  
```
