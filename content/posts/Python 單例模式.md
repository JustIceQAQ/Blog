---
title: 'Python 單例模式'
date: '2026-08-17T13:16:08+08:00'
lastmod: '2026-08-18T13:16:08+08:00'
draft: false
tags: [ "Python" , "設計模式", "單例模式" ]
slug: '/python-singleton-pattern/'
---

## 說明

- 單例模式的故事與背景可參考[^singleton]
- 建立物件時，若需保持該物件在記憶體中只有一個實例，藉此節省記憶體並免除重複初始化的開銷，例如：
    - 環境變數載入
    - 快取 / DB 連線類別
    - 需要全域維護狀態的工具物件

----

## 範例

### 1. 網路上常見的傳統作法 (`__new__`)

- 透過 _instance 來記錄 class 是否初始化過

```python
# demo_single.py
class DemoSingle:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, q, a):
        self.q = q
        self.a = a


ds1 = DemoSingle(1, 2)
ds2 = DemoSingle(1, 2)

ds1 is ds2
# True
```

---

### 2. 更 Pythonic 的作法

- 根據Python 官方文件[^how-do-i-share-global-variables-across-modules]的建議，利用 Python 模組在首次 import
  時只會執行並快取一次的特性，直接定義模組級物件

```python
# demo_single.py
class DemoSingle:
    def __init__(self, q, a):
        self.q = q
        self.a = a


# 直接在 demo_single 就初始化完成，後續不管其他 *.py 怎麼 import 該 ds_obj 物件皆為單例
ds_obj = DemoSingle(1, 2)

# a.py
from demo_single import ds_obj

...

# b.py
from demo_single import ds_obj

...
```

---

### 3. 利用 lru_cache 快取物件

- 如果不想用透過全域變數控制，也不想透過`__new__`新增，可以使用`lru_cache`來快取物件
- 既可達到單例化物件的效果，同時可降低理解程式時的認知負荷 (Cognitive Load[^Cognitive load])

```python
from functools import lru_cache


class DemoSingle:
    def __init__(self, q, a):
        self.q = q
        self.a = a


@lru_cache(maxsize=1)
def get_demo_single() -> DemoSingle:
    return DemoSingle(1, 2)

```

[^singleton]: https://python-patterns.guide/gang-of-four/singleton/
[^how-do-i-share-global-variables-across-modules]: https://docs.python.org/3/faq/programming.html#how-do-i-share-global-variables-across-modules
[^Cognitive load]:https://zh.wikipedia.org/zh-tw/%E8%AA%8D%E7%9F%A5%E8%B2%A0%E8%8D%B7
