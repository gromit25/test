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
conn = psycopg2.connect('host=192.168.35.66 port=5432 dbname=postgres user=postgres password=password',
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
