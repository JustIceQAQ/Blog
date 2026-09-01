---
title: 'Python 時間機器 time-machine'
date: '2026-09-01T18:44:58+08:00'
draft: false
tags: [ "Python", "time-machine" ]
categories: [ "Package", "Unittest" ]
slug: '/python-package-time-machine/'
---

## 說明

- time-machine[^time-machine]套件，如其名"時間機器"，透過套件方法可以影響`datetime.datetime.now`, `time.time`
  等Python內建時間方法。
    - 具體影響的 method 請參考 time-machine-mocked-functions [^time-machine-mocked-functions]
- 方便用於 unittest/pytest下與 日期/時間 相關的測試

----

## 範例

----

### 安裝

```shell
uv add time-machine fastapi
# or
# pip install time-machine fastapi
```

----

### Case 1

----

#### 功能測試

----

```python
# app.py
import datetime as dt

import uvicorn
from fastapi import FastAPI
import time_machine

app = FastAPI()


@app.get("/real-time")
async def realtime():
    """取得當下完整時間日期 API"""
    return {"realtime": dt.datetime.now()}


@app.get("/time-machine")
async def timemachine():
    """將時間設定在 2000-01-01，用 start 與 stop 控制 dt.datetime.now的行為 """
    traveller = time_machine.travel(dt.datetime(2000, 1, 1))
    traveller.start()
    result = {"timemachine": dt.datetime.now()}
    traveller.stop()
    return result


@app.get("/with-time-machine")
async def withtimemachine():
    """將時間設定在 1975-01-01，用 with 控制 dt.datetime.now的行為"""
    with time_machine.travel(dt.datetime(1975, 1, 1)):
        result = {"timemachine": dt.datetime.now()}
    return result


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)

```

----

#### 功能測試結果

----

```shell
curl -X 'GET' \
  'http://127.0.0.1:8000/real-time' \
  -H 'accept: application/json'

{
  "realtime": "2026-09-01T18:44:58"
}
```

```shell
curl -X 'GET' \
  'http://127.0.0.1:8000/time-machine' \
  -H 'accept: application/json'

{
  "timemachine": "2000-01-01T08:00:00"
}
```

```shell
curl -X 'GET' \
  'http://127.0.0.1:8000/with-time-machine' \
  -H 'accept: application/json'

{
  "timemachine": "1975-01-01T08:00:00"
}
```

- 透過 `time_machine.travel` start/stop 或是 with 寫法，可以立刻穿越時間

----

### Case 2

----

#### unittest

----

- 透過FastAPI的`Depends`功能設計兩組時間檢查機制，限制 API 在特定時間條件下才能使用。
- 並且透過 pytest 進行單元測試，藉以驗證`Depends`攔截是否能正常運行。

```python
# app2.py
import datetime as dt

import uvicorn
from fastapi import FastAPI, Depends, HTTPException, status

app = FastAPI(
    responses={
        status.HTTP_423_LOCKED: {"model": dict},
    },
)


async def datetime_2024_locked_depend():
    """若現在時間 大於 2024-01-01 則 return 423 status"""
    date_lock = dt.datetime(2024, 1, 1)
    now = dt.datetime.now()
    if now > date_lock:
        raise HTTPException(status.HTTP_423_LOCKED, f"datetime now is gt {date_lock:%Y-%m-%d}")


async def datetime_3000_locked_depend():
    """若現在時間 大於 3000-01-01 則 return 423 status"""
    date_lock = dt.datetime(3000, 1, 1)
    now = dt.datetime.now()
    if now > date_lock:
        raise HTTPException(status.HTTP_423_LOCKED, f"datetime now is gt {date_lock:%Y-%m-%d}")


@app.get("/just_use_before_2024_year", dependencies=[Depends(datetime_2024_locked_depend)])
async def just_use_before_2024_year():
    return {"😏": "👌"}


@app.get("/just_use_after_3000_year", dependencies=[Depends(datetime_3000_locked_depend)])
async def just_use_before_3000_year():
    return {"🧓": "👍"}


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8001)

```

----

#### unittest 結果

----

```python
# test_datelock.py
from fastapi.testclient import TestClient
from .app2 import app
import datetime as dt
import time_machine


def test_datelock_200():
    client = TestClient(app)
    with time_machine.travel(dt.datetime(2023, 1, 1)):
        response = client.get("/just_use_before_2024_year")
    assert response.status_code == 200

    response = client.get("/just_use_after_3000_year")
    assert response.status_code == 200


def test_datelock_423():
    client = TestClient(app)
    response = client.get("/just_use_before_2024_year")
    assert response.status_code == 423
    with time_machine.travel(dt.datetime(3001, 1, 1)):
        response = client.get("/just_use_after_3000_year")
    assert response.status_code == 423

```

```shell
============================= test session starts =============================
collecting ... collected 2 items

test_datelock.py::test_datelock_200 PASSED                               [ 50%]
test_datelock.py::test_datelock_423 PASSED                               [100%]

============================== 2 passed in 0.40s ==============================

Process finished with exit code 0
```

----

- 有關於時間相關條件的測試，可使用time_machine 來自動對 datetime 套件進行 mock方法

[^time-machine]: https://github.com/adamchainz/time-machine

[^time-machine-mocked-functions]: https://time-machine.readthedocs.io/en/latest/usage.html#mocked-functions