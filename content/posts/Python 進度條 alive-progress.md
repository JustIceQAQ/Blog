---
title: 'Python 進度條 - alive-progress'
date: '2026-08-20T10:50:35+08:00'
draft: false
tags: [ "Python", "Package", "alive-progress" ]
slug: '/python-package-alive-progress/'
---

## 說明

- 進度條 (Progress Bar)是一種透過視覺化手法，讓使用者知道目前執行的進度。 ~~目的其實是為了要安撫人心，大多都是假象~~
- 除了常見的tqdm[^tqdm]外，本次將來嘗試alive-progress[^alive-progress]

----

## 範例

----

### 安裝

```shell
pip install alive-progress
# or
uv add alive-progress
```

----

### 使用

----

#### Case 1 - 官方預設寫法

- 使用 with 來管理 bar 條的動作
    - 事前告知alive_bar 分母為 1000
    - 之後透過for loop，每執行完 time.sleep 後 執行一次 `bar()` 來推進一次分子

```python
from alive_progress import alive_bar
import time


def case_1():
    counts = 1000
    with alive_bar(counts) as bar:
        for _ in range(counts):
            time.sleep(.003)
            bar()


if __name__ == '__main__':
    case_1()
```

```shell
# 執行中
|██████████████████████▊                 | ▅▇▇ 569/1000 [57%] in 2s (~2s, 236.1/s)

# 執行結束 
|████████████████████████████████████████| 1000/1000 [100%] in 4.2s (237.98/s)
```

----

#### Case 2 - 簡便的寫法

- 載入 alive_progress 的 alive_it 方法後，並直接給予迭代的物件，剩下 for loop 即可處理自己需要執行的事情

```python
from alive_progress import alive_it
import time


def case_2():
    for _ in alive_it(range(100)):
        time.sleep(.03)
        pass


if __name__ == '__main__':
    case_2()
```

```shell
# 執行中
|████████████████████████████▍           | ▅▇▇ 71/100 [71%] in 2s (~1s, 32.1/s)

# 執行結束 
|████████████████████████████████████████| 100/100 [100%] in 3.2s (31.94/s)
```

----

#### Case 3 - 自定義進度的寫法

- 定義了 `step1`、`step2`和`step3`方法，這些方法是模擬實際執行時等待的時間，最後定義了一個 `run` 來執行
- 透過使用 logging.getLogger ("alive_progress") 來在console 顯示一些執行狀態

```python
from alive_progress import alive_bar, alive_it
import random
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("alive_progress")


class Case3:
    def get_random_sec(self) -> int:
        return random.randint(1, 5)

    def step1(self):
        time.sleep(self.get_random_sec())
        logger.info("step1 success")

    def step2(self):
        time.sleep(self.get_random_sec())
        logger.info("step2 success")

    def step3(self):
        time.sleep(self.get_random_sec())
        logger.info("step3 success")

    def run(self):
        with alive_bar(3) as bar:
            self.step1()
            bar()
            self.step2()
            bar()
            self.step3()
            bar()


if __name__ == '__main__':
    Case3().run()
```

```shell
on 0: INFO:alive_progress:step1 success
on 1: INFO:alive_progress:step2 success
on 2: INFO:alive_progress:step3 success
|████████████████████████████████████████| 3/3 [100%] in 7.0s (0.32/s)

```

----

#### Case Fun - 💩

- alive-progress 是一個靈活性高的進度條，若之後有開發 console 類型的程式 或是 寫一個 loop 的 script 都可以用這個套件來視覺化進度
- 若有必要也可以建立自己的 style bar

```python
from alive_progress.animations import bar_factory
from alive_progress import alive_bar
import time


def case_fun():
    my_bar = bar_factory("💩", tip="👈", borders=("🤮->|", "|<-🤬",))
    with alive_bar(1000, bar=my_bar) as bar:
        for _ in range(1000):
            time.sleep(.003)
            bar()
```

```shell
🤮->|💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩|<-🤬 1000/1000 [100%] in 3.8s (265.42/s)
```

[^tqdm]: https://github.com/tqdm/tqdm

[^alive-progress]: https://github.com/rsalmei/alive-progress