# InfluxDB

InfluxDB URL : https://www.influxdata.com/

## InfluxDB 

Influx DB란 많은 쓰기 작업과 쿼리 부하를 처리하기 위해 2013년에 Go 언어로 개발된 오픈소스 Time Series Database(시계열 데이터베이스). Influx DB는 많은 TSDB들(Prometheus, TimescaleDB, Graphite, 등) 중에서 가장 유명하고, 많이 사용되는 데이터베이스이다. Influx DB는 Distributed, Scale horizontally하게 설계되어 새로운 노드만 추가하면 손쉽게 scale-out할 수 있으며, Restful API를 제공하고 있어 API 통신이 가능하다.



### InfluxDB 설치


```bash
$ sudo curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
OK
$ sudo echo "deb https://repos.influxdata.com/ubuntu bionic stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
$ sudo apt update
$ sudo apt install influxdb
$ sudo systemctl unmask influxdb.service
$ sudo systemctl start influxdb
```
- TSM(Time Structured Merge tree)
InfluxDB를 위한 전용 데이터 저장 방식을 말합니다. TSM은 기존 B+ 혹은 LSM 트리 구현보다 더 높은 압축 및 쓰기/읽기 처리를 할 수 있습니다. 자세한 내용은 Storage Engine를 참고하세요.


### linux 날자 표시 변경


```bash
$ date --date='@1639958400'
Mon 20 Dec 2021 09:00:00 AM KST
$ date -d "2021-12-20T09:00:00" +%s
1639958400
```



### InfluxDB 설정


```bash
$ influx
Connected to http://localhost:8086 version 1.8.9
InfluxDB shell version: 1.8.9
>
> CREATE USER admin WITH PASSWORD 'EXAMPLE_PASSWORD' WITH ALL PRIVILEGES
> quit

$ sudo vi /etc/influxdb/influxdb.conf 
...
[http]
  ...
  # Determines whether user authentication is enabled over HTTP/HTTPS.
  auth-enabled = true
...
wq!
$ sudo systemctl restart influxdb
```



### InfluxDB Database 생성


```bash
$ influx
Connected to http://localhost:8086 version 1.8.9
InfluxDB shell version: 1.8.9
> SHOW DATABASES
> SHOW DATABASES
name: databases
name
----
_internal
atid1024
water_db
> use water_db
Using database water_db
> INSERT water_level,tank_location=MIAMI value_in_ft=10.00
> INSERT water_level,tank_location=SEATTLE value_in_ft=5.4
> INSERT water_level,tank_location=ATLANTA value_in_ft=0.5
> INSERT water_level,tank_location=DALLAS value_in_ft=3.2
> INSERT water_level,tank_location=LOS-ANGELES value_in_ft=1.3
```

- INSERT MEASUREMENT, TAG_1=VALUE_1, TAG_2=VALUE_2, TAG_N=VALUEN FIELD_1=VALUE_1, FIELD_2=VALUE_2, FIELD_N=VALUE_N
- 실제 운영환경에서는 설치된 지역의 sensor나 InfluxDB server의 API를 통해 data가 수신됩니다.



### InfluxDB bulk data 주입


```bash
# windows 에서 생성된 file의 line 끝 ^M 값 삭제
$ tr -d '\015' < pv-AI028.txt > sit-pv-ai028.txt
$ tr -d '\015' < pv-AI030.txt > sit-pv-ai028.txt
$ influx
Connected to http://localhost:8086 version 1.8.9
InfluxDB shell version: 1.8.9
> CREATE DATABASE SIT_PV
$ ls
sit-pv-ai028.txt sit-pv-ai030.txt
$ influx -import -path=sit-pv-ai028.txt -precision=s -database=SIT_PV -username=admin -password=wjdgid010
$ influx -import -path=sit-pv-ai030.txt -precision=s -database=SIT_PV -username=admin -password=wjdgid0103
$ influx -import -path=pv-AI028.txt -precision=s -database=SIT_PV

```



### InfluxDB Data 입력 및 조작


```sql
> use water_db
Using database water_db
> INSERT water_level,tank_location=MIAMI value_in_ft=10.00
> INSERT water_level,tank_location=SEATTLE value_in_ft=5.4
> INSERT water_level,tank_location=ATLANTA value_in_ft=0.5
> INSERT water_level,tank_location=DALLAS value_in_ft=3.2
> INSERT water_level,tank_location=LOS-ANGELES value_in_ft=1.3
> select "tank_location","value_in_ft" from "water_level"
name: water_level
time                tank_location value_in_ft
----                ------------- -----------
1640399607387426931 MIAMI         10
1640399614635189605 SEATTLE       5.4
1640399621643165027 ATLANTA       0.5
1640399628644464793 DALLAS        3.2
1640399635965012926 LOS-ANGELES   1.3
> SELECT "tank_location" FROM "water_level"
> SELECT "tank_location", "value_in_ft" FROM "water_level" WHERE tank_location='MIAMI'
name: water_level
time                tank_location value_in_ft
----                ------------- -----------
1640399607387426931 MIAMI         10

> DELETE FROM water_level WHERE tank_location='ATLANTA'
> SELECT "tank_location","value_in_ft" FROM "water_level"
name: water_level
time                tank_location value_in_ft
----                ------------- -----------
1640399607387426931 MIAMI         10
1640399614635189605 SEATTLE       5.4
1640399628644464793 DALLAS        3.2
1640399635965012926 LOS-ANGELES   1.3
> SELECT LAST("value_in_ft") FROM "water_level" WHERE "tank_location"='DALLAS'
name: water_level
time                last
----                ----
1640399628644464793 3.2
```

- INSERT MEASUREMENT, TAG_1=VALUE_1, TAG_2=VALUE_2, TAG_N=VALUEN FIELD_1=VALUE_1, FIELD_2=VALUE_2, FIELD_N=VALUE_N
- 실제 운영환경에서는 설치된 지역의 sensor나 InfluxDB server의 API를 통해 data가 수신됩니다.



### InfluxDB HTTP API 


```bash
$ curl -G 'http://localhost:8086/query?pretty=true' -u admin:wjdgid0103 --data-urlencode "db=sit" --data-urlencode "q=SELECT \"tank_location\", \"value_in_ft\" FROM \"water_level\""
{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "water_level",
                    "columns": [
                        "time",
                        "tank_location",
                        "value_in_ft"
                    ],
                    "values": [
                        [
                            "2021-12-25T02:33:27.387426931Z",
                            "MIAMI",
                            10
                        ],
                        [
                            "2021-12-25T02:33:34.635189605Z",
                            "SEATTLE",
                            5.4
                        ],
                        [
                            "2021-12-25T02:33:48.644464793Z",
                            "DALLAS",
                            3.2
                        ],
                        [
                            "2021-12-25T02:33:55.965012926Z",
                            "LOS-ANGELES",
                            1.3
                        ]
                    ]
                }
            ]
        }
    ]
}
```
```bash
atid@redolence:~$ curl -G 'http://localhost:8086/query?pretty=true' -u admin:wjdgid0103 --data-urlencode "db=statistics" --data-urlencode "q=SELECT * FROM \"AI028_1H\" WHERE time = 1640174400000000000"{
    "results": [
        {
            "statement_id": 0,
            "series": [
                {
                    "name": "AI028_1H",
                    "columns": [
                        "time",
                        "PVEF",
                        "SIRS",
                        "TAE"
                    ],
                    "values": [
                        [
                            "2021-12-22T12:00:00Z",
                            100.0528871744839,
                            730.8009999999999,
                            731.1875
                        ]
                    ]
                }
            ]
        }
    ]
}
```
```bash
atid@redolence:~$ date --date='@1640188800'
Wed 22 Dec 2021 04:00:00 PM UTC

1640174400 
1640178000 
1640181600 
1640185200 
1640188800 
```


### InfluxDB VS RDB 비교

|    관계형 DB     |       InfluxDB        |
| :--------------: | :-------------------: |
|     database     |       database        |
|      table       |      measurement      |
|      column      |          Key          |
|  indexed column  | tag key (string only) |
| unindexed column |       field key       |
|       row        |         point         |

- point : 단일 행의 레코드를 가진 SQL 데이터베이스와 유사, 단일 필드 모음으로 구성되고 각 point는 그자체로 고유하게 식별
- timestamp : point에 관련된 날짜와 시간, InfluxDB의 timestamp는 항상 나노세컨드 단위의 unix time 값으로 저장 (ex. 1639958460000000000ns = 2021-12-20T09:01)
- line protocol : InfluxDB에 point를 쓰기위한 텍스트 기반 형식

보존정책 생성

### InfluxDB Retention Policy (RP: 보존정책)

데이터를 사용할 수 있는 기간 혹은 시간을 지정

- duration: InfluxDB가 데이터를 보관하는 기간
- replication factor: 클러스터에 저장된 데이터의 복사본 수
- shared groups : 샤드와 샤드의 논리적 컨테이너로 실제 데이터가 들어 있음
- shard group duration : 샤드 그룹에 의해 보호되는 시간 범위


```bash
> show retention policies on atid1024
name    duration shardGroupDuration replicaN default
----    -------- ------------------ -------- -------
autogen 0s       168h0m0s           1        true
> 
```

- show retention policies on <database name>
- atid1024 데이터베이스의 retention policy를 조회
- RP가 정의되지 않은 경우 기본 RP(autogen)가 적용
- autogen은 복제 수가 1이고 샤드그룹 기간이 7일이고 무한대의 지속기간을 가짐

보존정책과 샤드

| Retention Policy's Duration | Shard Group Duration |
| :-------------------------- | :------------------- |
| < 2 days                    | 1 hour               |
| >= 2 days and <= 6 months   | 1 day                |
| > 6 months                  | 7 days               |



보존정책 생성


```bash
> create retention policy sit_rp on atid1024 duration 3d replication 1 shard duration 1d default
> show retention policies 
name    duration shardGroupDuration replicaN default
----    -------- ------------------ -------- -------
autogen 0s       168h0m0s           1        false
sit_rp  72h0m0s  24h0m0s            1        true
> 
```

- rp 이름은 sit_rp, database는 atid1024, duration 3일 replicas 는 1개 샤드 주기는 1주



보존정책 변경


```bash
> alter retention policy sit_rp on atid1024 duration 2d shard duration 1d default
> show retention policies
name    duration shardGroupDuration replicaN default
----    -------- ------------------ -------- -------
autogen 0s       168h0m0s           1        false
sit_rp  48h0m0s  24h0m0s            1        true
> 
```

- rp 이름인 sit_rp의 값을 database 이름이 atid1024에 대해 duration 2일로 변경



보존정책 삭제


```bash
> drop retention policy sit_rp on atid1024
> show retention policies
name    duration shardGroupDuration replicaN default
----    -------- ------------------ -------- -------
autogen 0s       168h0m0s           1        false
> 
> alter retention policy autogen on atid1024 default
> show retention policies on atid1024
name    duration shardGroupDuration replicaN default
----    -------- ------------------ -------- -------
autogen 0s       168h0m0s           1        true
> 
```

- rp 이름인 sit_rp의 값을 삭제 

- autogen을 default로 설정 (false -> true)

  

### InfluxQL (classic InfluxDB query)

InfluxQL query language 사용시

| Name                | Description                                                  |
| :------------------ | :----------------------------------------------------------- |
| `Name`              | The data source name. This is how you refer to the data source in panels and queries. We recommend something like `InfluxDB-InfluxQL`. |
| `Default`           | Default data source means that it will be pre-selected for new panels. |
| `URL`               | The HTTP protocol, IP address and port of your InfluxDB API. InfluxDB API port is by default 8086. |
| `Access`            | Server (default) = URL needs to be accessible from the Grafana backend/server, Browser = URL needs to be accessible from the browser. **Note**: Browser (direct) access is deprecated and will be removed in a future release. |
| `Allowed cookies`   | Cookies that will be forwarded to the data source. All other cookies will be deleted. |
| `Database`          | The ID of the bucket you want to query from, copied from the [Buckets page](https://docs.influxdata.com/influxdb/v2.0/organizations/buckets/view-buckets/) of the InfluxDB UI. |
| `User`              | The username you use to sign into InfluxDB.                  |
| `Password`          | The token you use to query the bucket above, copied from the [Tokens page](https://docs.influxdata.com/influxdb/v2.0/security/tokens/view-tokens/) of the InfluxDB UI. |
| `HTTP mode`         | How to query the database (`GET` or `POST` HTTP verb). The `POST` verb allows heavy queries that would return an error using the `GET` verb. Default is `GET`. |
| `Min time interval` | (Optional) Refer to [Min time interval](https://grafana.com/docs/grafana/latest/datasources/influxdb/#min-time-interval). |
| `Max series`        | (Optional) Limits the number of series/tables that Grafana processes. Lower this number to prevent abuse, and increase it if you have lots of small time series and not all are shown. Defaults to 1000. |
