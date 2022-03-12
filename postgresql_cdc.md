```
FROM postgres:bullseye
RUN apt-get update && apt-get install -y --no-install-recommends postgresql-14-wal2json
```

```
docker build . -t my-postgres:0.1
```

-------------------------------------------------

```
version: '2'

#
networks:
  stocks_net:

#
services:
  db:
    image: my-postgres:0.1
    container_name: postgres
    restart: always
    ports:
      - 5432:5432
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - ./postgres_data/:/var/lib/postgresql/data
    command:
      - "postgres"
      - "-c"
      - "wal_level=logical"
      - "-c"
      - "shared_preload_libraries=wal2json"
    networks:
      - stocks_net
```
    
--------------------------
    
```
show max_replication_slots;
show wal_level;
```

test_decoder   
wal2json    

```
select pg_create_logical_replication_slot('slot_test', 'wal2json');
```
    
```
insert into "TestSchema".app_config values('acqusition', 'from_time', '09:00');
commit;

select pg_logical_slot_peek_changes('slot_test', NULL, NULL);
```

```
select *   
from pg_replication_slots   
where slot_name = 'slot_test';   
```
    
```
select FROM pg_drop_replication_slot('slot_test');
```
    
https://dev.to/thiagosilvaf/how-to-use-change-database-capture-cdc-in-postgres-37b8    
https://www.psycopg.org/docs/extras.html    
    
--------------------------

```
pip install psycopg2
```
    
https://wiki.postgresql.org/images/1/1b/Valentine_Gogichashvili_-_psycopg2_Replication.pdf    

```python
import psycopg2
from psycopg2.extras import LogicalReplicationConnection

# postgresql db와 연결
# "with psycopg2.connect(..., connection_factory=LogicalReplicationConnection) as conn:"
# 형태로 사용하면 안됨 -> 이럴 경우, transaction block으로 인식하게 되어 아래의 오류 메시지가 출력됨
# create_replication_slot 함수에만 적용됨
# start_replication, consume_stream는 상관 없음
# error msg:
# "CREATE_REPLICATION_SLOT ... LOGICAL exports a snapshot. If it was called outside of transaction the snapshot should be cleared here."
conn = psycopg2.connect('host=111.111.111.111 port=5432 dbname=postgres user=postgres password=password',
                        connection_factory=LogicalReplicationConnection)
cur = conn.cursor()

cur.create_replication_slot('slot_test', output_plugin='wal2json')
print('create slot')

cur.start_replication(slot_name='slot_test', decode=True)


def consume(msg):
    print(type(msg))
    print(msg.payload)


cur.consume_stream(consume)
```


```python
import psycopg2
from psycopg2.extras import LogicalReplicationConnection

import select
import threading
from datetime import datetime
import time

import json


def wait(conn):
    while True:
        state = conn.poll()
        if state == psycopg2.extensions.POLL_OK:
            break
        elif state == psycopg2.extensions.POLL_WRITE:
            select.select([], [conn.fileno()], [])
        elif state == psycopg2.extensions.POLL_READ:
            select.select([conn.fileno()], [], [])
        else:
            raise psycopg2.OperationalError("poll() returned %s" % state)


config_monitoring_slot_name = 'config_monitoring_slot'
dsn = 'host=111.111.111.111 port=5432 dbname=postgres user=postgres password=password'

aconn = psycopg2.connect(dsn, connection_factory=LogicalReplicationConnection, async_=True)
wait(aconn)
acur = aconn.cursor()

# slot을 생성함(메시지는 json 포맷으로 수신함)
acur.create_replication_slot(config_monitoring_slot_name, output_plugin='wal2json')
wait(aconn)
print('slot is created.')

acur.start_replication(slot_name=config_monitoring_slot_name, decode=True)
wait(aconn)
print(f'debug: {type(acur)}')


is_continue = True


def consume(cur):
    while is_continue:

        msg = cur.read_message()

        if msg is not None:
            cdc_json = json.loads(msg.payload)
            print(cdc_json['change'][0]['schema'])
            print(cdc_json['change'][0]['table'])
        else:
            #
            now = datetime.now()
            timeout = 10.0 - (now - cur.io_timestamp).total_seconds()
            sel = select.select([cur], [], [], max(0, timeout))
            if not any(sel):
                cur.send_feedback()  # timed out, send a keepalive message


t1 = threading.Thread(target=consume, args=(acur,))
t1.start()

print('start wait')
time.sleep(10)

print('change is_continue to False')
#is_continue = False
```
