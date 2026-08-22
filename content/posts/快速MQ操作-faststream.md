---
title: '快速MQ操作 - faststream'
date: '2026-08-24T12:49:27+08:00'
draft: false
tags: [ "Python", "faststream", "fastapi" ]
categories: [ "Package", "Message Queue" ]
slug: '/python-package-faststream/'
---

## 說明

- faststream[^faststream]簡化了訂閱者 (Subscriber) 與推送者 (Publisher) Message Queue (下稱 MQ)
  常見操作，提供統一的API規格，讓使用者可以簡易的操作各類型MQ框架。
    - 目前支援Kafka, RabbitMQ, NATS, Redis 與 MQTT等 MQ框架
- 支援自動化文件 (需另外下cli執行)

----

## 範例

- 本文使用 docker 快速建立 redis 服務，在此不特別說明建置方法

```shell
pip install faststream[redis,cli]
# or
uv add faststream[redis,cli]
```

----

### Server 端 - faststream_server.py

- 透過 pydantic 建立格式化payload，並寫好兩個Topic來做資料接收與 def 之間的資料溝通

```python
# faststream_server.py
from typing import Literal

from faststream import FastStream
from faststream.redis import RedisBroker
from pydantic import BaseModel

broker = RedisBroker("redis://localhost:6378/")

app = FastStream(broker)


class PolitePayload(BaseModel):
    name: str
    say: Literal["PLS", "THX", "SRY"]


@broker.subscriber("topic:agent-bot")
@broker.publisher("topic:Bob")
async def agent_handle_msg(data: PolitePayload) -> PolitePayload:
    print(f"publisher {data.name} messages")
    return data


@broker.subscriber("topic:Bob")
async def bob_func(data: PolitePayload):
    print(data)
```

----

### Server 端 - 服務啟動

```shell
uv run faststream run faststream_server:app
2026-08-20 13:55:57,740 INFO     - FastStream app starting...
2026-08-20 13:55:57,749 INFO     - topic:agent-bot |            - `AgentHandleMsg` waiting for messages
2026-08-20 13:55:57,753 INFO     - topic:Bob       |            - `BobFunc` waiting for messages
2026-08-20 13:55:57,754 INFO     - FastStream app started successfully! To exit, press CTRL+C
```

----

### Client 端 - say_something.py

- 簡單寫個 `say_correct_format` 與 `say_error_format` def，除了訊息傳遞是否正常外，也測試是否能驗證資料格式

```python
# say_something.py

import asyncio

from faststream.redis import RedisBroker

broker = RedisBroker("redis://localhost:6378/")


async def say_correct_format(topic: str, name: str, say: str):
    async with broker as br:
        await br.publish({
            "name": name,
            "say": say,
        }, topic)


async def say_error_format(topic: str, name: str, say: str):
    async with broker as br:
        await br.publish({
            "name": name,
            "eat": say,
        }, topic)


async def main():
    await say_correct_format("topic:agent-bot", "William", "PLS")
    # await say_error_format("topic:agent-bot", "William", "PLS")


if __name__ == "__main__":
    asyncio.run(main())


```

----

### Client 端 - 發送正確格式訊息

```shell
uv run .\say_something.py

# faststream_server.py console
2026-08-20 13:58:27,464 INFO     - topic:agent-bot | 53c673ad-d - Received publisher William messages
2026-08-20 13:58:27,468 INFO     - topic:agent-bot | 53c673ad-d - Processed
2026-08-20 13:58:27,468 INFO     - topic:Bob       | 11c3a380-2 - Received name='William' say='PLS'
2026-08-20 13:58:27,469 INFO     - topic:Bob       | 11c3a380-2 - Processed
```

----

### Client 端 - 發送錯誤格式訊息

```python
# say_something.py
async def main():
    # await say_correct_format("topic:agent-bot", "William", "PLS")
    await say_error_format("topic:agent-bot", "William", "PLS")
```

```shell
uv run .\say_something.py

# faststream_server.py console
2026-08-20 14:00:53,828 ERROR    - topic:agent-bot | 39ce262e-0 - ValidationError: 1 validation error for agent_handle_msg
data.say
  Field required [type=missing, input_value={'name': 'William', 'eat': 'PLS'}, input_type=dict]
    For further information visit https://errors.pydantic.dev/2.13/v/missing
Traceback (most recent call last):
  ...
pydantic_core._pydantic_core.ValidationError: 1 validation error for agent_handle_msg
data.say
  Field required [type=missing, input_value={'name': 'William', 'eat': 'PLS'}, input_type=dict]
    For further information visit https://errors.pydantic.dev/2.13/v/missing
2026-08-20 14:00:53,831 INFO     - topic:agent-bot | 39ce262e-0 - Processed
```

### Server 端 - 產生自動化文件

```shell
faststream docs serve faststream_server:app

2026-08-20 14:04:20,500 INFO     - HTTPServer running on http://localhost:8000 (Press CTRL+C to quit)
2026-08-20 14:04:20,500 WARNING  - Please, do not use it in production.

```

[^faststream]: https://github.com/ag2ai/faststream