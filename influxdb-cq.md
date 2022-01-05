## influxData Down sampling
오랜 시간 동안 많은 양의 데이터로 작업하면 스토리지 문제가 발생할 수 있습니다. 자연스러운 해결책은 데이터를 다운 샘플링하는 것입니다. 고정밀 미가공 데이터를 제한된 시간 동안 유지하고 더 낮은 정밀도로 요약 된 데이터를 훨씬 더 오래 또는 영원히 저장하십시오.
InfluxDB는 데이터 다운 샘플링 및 오래된 데이터 만료 프로세스를 자동화하는 CQ (Continuous Queries) 및 RP (Retention Policies)의 두 가지 기능을 제공합니다.

### Definitions
- 연속 쿼리 (CQ)는 데이터베이스 내에서 자동으로 주기적으로 실행되는 InfluxQL 쿼리입니다. CQ는 SELECT 절에 함수가 필요하며 GROUP BY time() 절을 포함해야합니다.
- 보존 정책 (RP)는 InfluxDB 데이터를 유지하는 시간에 대한 설명 InfluxDB의 데이터 구조의 일부입니다. InfluxDB는 로컬 서버의 타임 스탬프를 데이터의 타임 스탬프와 비교하고 RP의 DURATION 보다 오래된 데이터를 삭제합니다 . 단일 데이터베이스에는 여러 개의 RP가있을 수 있으며 RP는 데이터베이스마다 고유합니다.

### Sample Data
```text
```

### Goal
장기적으로는 1시간 간격으로 전화 각 장치에 대한 값에만 관심이 있다고 가정합니다.  다음 단계에서는 RP 및 CQ를 사용하여 다음을 수행합니다.
- 1분 해상도 data를 1시간 해상도 data로 자동 집계
- 2시간 보다 오래된 원시 1분 해상도 데이터 자동 삭제
- 15주가 지난 1시간 해상도 data 자동 삭제

### Database 준비
데이터베이스 sit_pv에 데이터를 쓰기 전에 다음 단계를 수행합니다.  CQ는 최근 데이터에 대해서만 실행되므로 데이터를 적재 하기 전에 이 작업을 수행 합니다. 오래된 타임 스탬프와 데이터 now()를 뺀 FOR CQ의 절 또는 now() fmf 뺀 GROUP BY time() CQ가 없는 경우 간격 FOR 절.
1. 데이터베이스 생성
- 데이터베이스에 포인트를 쓸 때 명시적 RP를 제공하지 않으면 InfluxDB는 DEFAULT RP에 씁니다.  1분 고해상도 데이터를 기록하기 원하기 때문에, 2 시간 동안 데이터를 유지하는 RP 생성
  ```
  >  CREATE RETENTION POLICY "two_hours" ON "food_data" DURATION 2h REPLICATION 1 DEFAULT
  ```
  -- food_data 데이터베이스에 two_hours RP 생성
  -- two_hours는 2시간(2h) 동안 데이터를 유지하며 food_data 데이터베이스의 DEFAULT RP 입니다.
  -- REPLICATION 1 은 필수 매개 변수이지만 단일 노드 인스턴스의 경우 항상 1로 설정해야합니다.
2.  15주 RP 생성
- 다음으로 15주 동안 데이터를 유지하고 DEFAULT가 아닌 다른 RP를 생성하려고합니다. 궁극적으로 2시간 롤업 데이터가 이 RP에 저장됩니다.
  ```
  > CREATE RETENTION POLICY "a_year" ON "food_data" DURATION 52w REPLICATION 1
  ```
3. CQ(Continuous Query) 생성
- 이제 1분 해상도를 1시간 해상도로 자동 및 주기적으로 다운 샘플링하고 그 결과를 다른 보존 정책 및 다른 측정 값으로 저장하는 CQ를 생성합니다
  ```
  > CREATE CONTINUOUS QUERY "cq_30m" ON "food_data" BEGIN
    SELECT mean("website") AS "mean_website",mean("phone") AS "mean_phone"
    INTO "a_year"."downsampled_orders"
    FROM "orders"
    GROUP BY time(30m)
    END
  ```
4. 새로운 CQ와 두 개의 새로운 RP로 food_data는 데이터 수신을 시작할 준비가 되었습니다.  데이터베이스에 데이터를 쓰고 약간의 작업을 수행 한 후 orders와 downsampled_orders의 두 가지 측정을 볼 수 있습니다.
  ```text
  > SELECT * FROM "orders" LIMIT 5
name: orders
---------
time			                phone  website
2016-05-13T23:00:00Z	  10     30
2016-05-13T23:00:10Z	  12     39
2016-05-13T23:00:20Z	  11     56
2016-05-13T23:00:30Z	  8      34
2016-05-13T23:00:40Z	  17     32

> SELECT * FROM "a_year"."downsampled_orders" LIMIT 5
name: downsampled_orders
---------------------
time			                mean_phone  mean_website
2016-05-13T15:00:00Z	  12          23
2016-05-13T15:30:00Z	  13          32
2016-05-13T16:00:00Z	  19          21
2016-05-13T16:30:00Z	  3           26
2016-05-13T17:00:00Z	  4           23
  ```
