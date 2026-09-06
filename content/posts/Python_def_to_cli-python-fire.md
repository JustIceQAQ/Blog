---
title: 'Python def to cli python-fire'
date: '2026-09-06T12:54:43+08:00'
draft: false
tags: [ "Python", "python-fire" ]
categories: [ "Package" ]
slug: '/python-package-fire/'
---

## 說明

- python-fire[^python-fire]，這是一個將Python def 或 class 的 method 快速直接轉換成cli 命令列程式使用。

----

## 範例

### 安裝

```shell
uv add fire
# or 
# pip install fire
```

----

### 官網範例 - hello.py

```python
# hello.py
import fire


def hello(name: str = "World"):
    return f"Hello {name}!"


if __name__ == '__main__':
    fire.Fire(hello)
```

```shell
uv run .\hello.py
> Hello World!

uv run .\hello.py -h
> INFO: Showing help with the command 'hello.py -- --help'.
> NAME
>     hello.py
> 
> SYNOPSIS
>     hello.py <flags>
> 
> FLAGS
>     -n, --name=NAME
>         Type: str
>         Default: 'World'

uv run .\hello.py -n QAQ
> Hello QAQ!

uv run .\hello.py -name QAQ
> Hello QAQ!
```

----

### 修改自官網範例 - friendly.py

```python
# friendly.py
import fire


class FriendlyCode:
    """
    有禮貌的程式，他會使用 請、謝謝、對不起
    """

    def say_please(self, name: str | None = "QAQ"):
        return f"Please, bestow upon us your boundless knowledge, oh {name}, the oracle of all realms."

    def say_thanks(self, name: str | None = "QAQ"):
        return (f"Thanks ever so much for sharing your unparalleled brilliance, {name}; "
                f"we're all just bystanders in the spectacle of your intellect.")

    def say_sorry(self, name: str | None = "QAQ"):
        return (f"Oh, sorry for intruding on your moment of enlightenment, {name}; "
                f"we momentarily forgot we were merely extras in the grand production of your life.")


if __name__ == '__main__':
    fire.Fire(FriendlyCode)
```

- 這樣就能執行子 cli 了，而且若有寫註釋，套件也能正常讀取

```shell
uv run .\friendly.py say_please -n QAQ
> Please, bestow upon us your ....

uv run .\friendly.py say_thanks -n QAQ
> Thanks ever so much for sharing ...

uv run .\friendly.py say_sorry -n QAQ 
> Oh, sorry for intruding on your moment of enlightenment, QAQ...
```

[^python-fire]:https://github.com/google/python-fire