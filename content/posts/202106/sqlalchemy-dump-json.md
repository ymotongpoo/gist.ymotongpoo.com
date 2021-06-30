---
title: "SQLAlchemyで結果をJSONにダンプする"
date: 2021-06-30T11:43:40+09:00
draft: false
tags: ["python", "sqlalchemy", "postgresql", "orm", "json"]
---

* SQLAlchemy: v1.4

## ORM APIを使ってデータを取得する

まずこういうようなコードがあったとする。


```python
from datetime import datetime

import sqlalchemy
from sqlalchemy import Column, DateTime, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class User(Base):
    __tablename__ == "users"
    id = Column(Integer(), autoincrement=True, primary_key=True)
    name = Column(String(), nullable=False)
    age = Column(Integer(), nullable=False)
    birthday = Column(DateTime(), nullable=False)

    def __init__(self, id: int, name: str, age: int, birthday: datetime) -> None:
        self.id = id
        self.name = name
        self.age = age
        self.birthday = birthday

db_config = {
    "pool_size": 5,
    "max_overflow": 2,
    "pool_timeout": 30,
    "pool_recycle": 1800,
}

engine = sqlalchemy.create_engine(
    sqlalchemy.engine.url.URL.create(
        drivername="postgreqsl+psycopg2", # pip install psycopg-binary for testing
        username="postgres",
        password="xxxxxxxxxxxxx",
        host="XX.XX.XX.XXX",
        port="5432",
        database="user_db",
    ),
    **db_config
)
engine.dialect.desription_encoding = None

factory = sessionmaker(bind=engine)
session = factory()
users = sesion.query().all() # このusersをJSON化して返したい
```

昔のバージョンでは次のようにしたらJSON化して返せた。

```python
...
dict_users = [user._asdict() for user in users]
return json.dumps(dict_users)
```

でもこの `_asdict()` はdeprecatedになったし、そもそもプライベートなメソッドなので使えない。
Userのスキーマはわかっているので、地道にデータ変換をして返すことにした。

```python
...
users = session.query().all()
return [dict_user(user) for user in users]

def json_user(user: User) -> list[dict[str, str]]:
    return {
        "id": user.id,
        "name": user.name,
        "age": user.age,
        "birthday": user.birthday.strftime("%Y-%m-%dT%H:%M:%SZ")
    }
```

## ORM APIを使わない場合

ORM APIを使わない場合は出来るっぽい。

```python
import sqlalchemy
from sqlalchemy import Column, Integer, MetaData
import json

engine = sqlalchemy.create_engine(
    ...略...
)

metadata = MetaData()
t = sa.Table(
    "foo",
    metadata,
    Column("id", Integer(), primary_key=True, autoincrement=True),
    Column("a", Integer()),
    Column("b", Integer()),
)
metadata.create_all(bind=engine)

engine.execute(t.insert().values(a=1, b=2))
engine.execute(t.insert().values(a=3, b=4))
engine.commit()

result = engine.execute(t.select()).fetchall()
data = json.dumps([dict(row.items()) for row in rsult])
```