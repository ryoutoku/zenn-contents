---
title: "超個人的!!Pythonにおけるバグになりやすい実装"
emoji: "💬"
type: "tech"
topics: ["Python"]
published: true
---

## はじめに

プロジェクトに入るたび、「アイエエエ!? ジッソウ!? ジッソウナンデ!?」という実装に出会う事が多いので、他山の石として。

それっぽく動くが、実装者が想定していないであろう動作をしそうな、バグになりそうな実装をまとめる。

Python ベースのサンプルコードだが、他言語一般でも同様に考えることができる。

## メソッドのデフォルト引数にオブジェクト型を指定する

ここの「オブジェクト型」は`int`型など**ではない**以下のようなオブジェクト型である。

- list
- dict
- その他、クラスとか

### 問題

初期化されていると思っていた変数が、初期化されずに使いまわされる。

### 対策案

デフォルト引数に`list`などのオブジェクト型を指定しない。
必要であれば、method 等のスコープ内で再定義する。

### サンプルコード

```python
from datetime import datetime

# デフォルト引数として空のlistを指定
def test_1(arg = []):
  arg.append('data')
  print(arg)

# 定義的には、test_1をcallする毎にargが初期化されそうだが、
# argの初期化が初回callの時だけ、かつ使いまわされるため、test_1をcallするたびに"data"が追加される
test_1()  # => ['data']
test_1()  # => ['data', 'data']

##############################################################################

def test_2(arg = datetime.now()):
    print(arg)

# argが保持されたままであるため、同じ時間が表示される
test_2()  # => 2022-09-01 23:40:52.386274
test_2()  # => 2022-09-01 23:40:52.386274
```

## 環境変数をデフォルト値ありで取得する

### 発生しうる問題

初期値として要求する環境変数を設定していないのにエラーとならないため、設定漏れやミスに気づきにくい。
実行環境ごとに環境変数から値を取得するといった場合に問題が発生する。
例えば、`product`環境実行時に環境変数の設定漏れがあったが、デフォルト値として`develop`環境の値を設定していた、など。

### 対策案

起動できない方がエラーに気付けるため、大人しくエラーとする。

### サンプルコード

```python
import os

# 悪い例 1
# ENV_BUCKETの設定漏れをしていた場合、(develop環境想定の)'local_path'になってしまう
BUCKET = os.environ.get('ENV_BUCKET', 'local_path')

# 悪い例 2
# ENV_BUCKETの設定漏れをしていた場合、BUCKETはNoneになる
# BUCKETが使われるときに初めて未設定である、という事が分かるため気付くのに時間がかかる
BUCKET = os.environ.get('ENV_BUCKET')

##############################################################################

# 改善案
# AWS_BUCKETの設定漏れをしていた場合エラーにすることで、設定ミスに気づける
BUCKET = os.environ['ENV_BUCKET']
```

## 並列処理として Process を使うか Thread を使うか検証しない

並列処理として thread を使った場合、意図した並列処理できない可能性がある。
サンプルコードでは、thread を使った場合だと並列処理できていないことが分かる。
細かいことはここら辺が参考になる。
https://qiita.com/nyax/items/659b07cd755f2ced563f

### 発生しうる問題

意図した並列処理できておらず、処理速度向上ができない。

### 対策案

GIL を考慮し、Thread を使うか Process を使うかを判断する必要がある。

### サンプルコード

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import time

def run():
    n = 10000000
    while n > 0:
        n -= 1

def thread_run(i:int):
    start = time.time()
    with ThreadPoolExecutor() as executor:
        for i in range(i):
            executor.submit(run)
    end = time.time()
    print(f'thread run:{i} time:{end - start}')

def process_run(i:int):
    start = time.time()
    with ProcessPoolExecutor() as executor:
        for i in range(i):
            executor.submit(run)
    end = time.time()
    print(f'process run:{i} time:{end - start}')

thread_run(1)
thread_run(5)
process_run(1)
process_run(5)

####
# main実行結果参考
# thread run:0 time:0.5941951274871826
# thread run:4 time:3.6495440006256104
# process run:0 time:0.6773443222045898
# process run:4 time:0.6274311542510986
```

## ループ処理をする際、for 文とリスト内包表記と built in function を検証しない

Python のループ処理の速度は for 文よりもリスト内包表記の方が早い。
また、for 文を書くよりも built in function を使う方が明瞭にかける。
実際にオブジェクトの構成やリスト長などにも依存するため、速度/可読性を考慮し検証する必要がある。

### 発生しうる問題

処理速度のボトルネックになりうる。

### 対策案

単純に「これが答え」ということができないため、処理内容を考慮したサンプルコードなどを作成、検証にて手段を選択する。
また、そもそものロジックを見直す方が根本的な解決になることもある。

### サンプルコード

1 から 100000 の中で、2 の倍数の値の合計値を出すサンプルコードを実行。
方法は以下 3 パターン。

- for 文を使う
- List 内包表記 + built in function(`sum`)を使う
- `range`をちょっと考えて for 文を使う

```python
from time import perf_counter

iter = 100000

########################
def test_1():
  # for文を使う
  start = perf_counter()
  count = 0
  for i in range(iter):
      if i % 2 == 0:
          count += i
  print(f'{perf_counter() - start:.8f}')

########################
def test_2():
  # list内包表記を使う
  start = perf_counter()
  tmp = [x for x in range(iter) if x % 2 == 0]
  print(f'{perf_counter() - start:.8f}')

########################
def test_3():
  # そもそもrangeの指定を変えてfor文を使う
  start = perf_counter()
  count = 0
  for i in range(0, iter, 2):
      count += i
  print(f'{perf_counter() - start:.8f}')

# ----------------------

# 実行結果
test_1()  # => 0.00932850
test_2()  # => 0.00687620
test_3()  # => 0.00356630
```

## 即時評価と遅延評価を考慮しない

コードの実装方法によって、即時評価と遅延評価と異なる方法で表現できるため、意識して実装する必要がある。
例えば、yield を使うと iterator / generator になり、遅延評価となる。
個人的に iterator をイメージしやすいのは`open()`を使ったファイルポインタ。`readline()`は一行ずつ(=評価されたとき)メモリ展開される。

### 発生しうる問題

重い処理のボトルネック調査が正しくできない。

### 対策案

iterator / generator 、即時評価と遅延評価を理解して使う。

### サンプルコード

以下 2 つの関数を作成し、処理時間を計測する。関数内では、重い処理を想定し`sleep`を噛ます。

- list を生成して返す関数(`return_list`。即時実行)
- iterator を返す関数(`return_iter`。遅延実行)

  - iterator を list 化(評価)し、そのタイミングで実際の処理が実行されることを確認

```python
from time import sleep

def return_list(list_size: int):
    # listを生成して返す
    # 検証用にsleepを噛ます
    return [(x, sleep(1)) for x in range(list_size)]

def return_iter(list_size: int):
    # iteratorとして数値を返す
    # 検証用にsleepを噛ます
    for i in range(list_size):
        sleep(1)
        yield i

list_size = 5

########################
def test_1():
  # return list
  start = perf_counter()
  create_list = return_list(list_size)
  print(f'{perf_counter() - start:.8f}', 'call return_list')


########################
def test_2():
  # return iterator
  # iteratorが返ってくるだけのため、時間がかからない
  start = perf_counter()
  create_iter = return_iter(list_size)
  print(f'{perf_counter() - start:.8f}', 'call return_iter')

  # iteratorをlist化。実際に評価する。ここで初めてsleep処理が実行される
  tmp_list = list(create_iter)
  print(f'{perf_counter() - start:.8f}', 'iter to list')


# 実行結果
test_1()
test_2()
# 5.00746500 call return_list
# 0.00000650 call return_iter
# 5.00681550 iter to list
```

## method などの引数に dict を使う

引数に`dict`を使うと、`dict`に含まれる`key`を把握しておかないと実装できない。

### 発生しうる問題

以下のようなときに問題が発生する。

- method を変更する時、どのような`key`があるのか、また対応する`value`の型が何か把握し辛い
  - `value`の型を固定できないため、どのような型のデータが入っているかは実装者に依存する
  - 把握するために call する側の処理を確認する必要がある
- method を call する時、どのような`key`が必要なのか method 内のロジックを確認する必要が出てくる

### 対策案

- 引数に`dict`は使わない
  - 必須の引数は明示的に定義する
  - 必須でない引数の場合は`dict`(`**kwargs`)を許容する (※独断と偏見ではある)

### バグサンプルコード

```python

# 悪い例 1
def sample_NG(d: dict):
  # dict内のkeyである'a','b'の存在を把握しておく必要がある
  a = d['a']
  b = d['b']

######################################
# 改善案 1
def sample_OK_1(a: int, b: str):
    # 引数を明示する
    ...

# 改善案 2
def sample_OK_2(a: int, b: str, **kwargs):
    """
    **kwargsはdictだが、必須でない
    """
    ...

# sample_OK_2のcallのしかた
data = {
  'a': 1,
  'b': 'b',
  'c': 3,
  'd': 4
}
sample_OK_2(**data)   # => a, b, {'c': 3, 'd': 4}　で引数が展開される
```

## method に dict を渡し method 内で dict を構築(変更)する

いわゆるトランザクションスクリプトにしない、ということ。
`boto3`の method の call 時の引数など、複雑な`dict`を構築する際は、method 内で`key`、`value`を追加しない。

### 発生しうる問題

以下問題が発生する。

- 可読性が低くなる
- テストしづらい
- `key`の上書きなども発生する可能性がある

### 対策案

method で`value`を返すようにし、`key`と`value`の定義は上位で設定する。

### バグサンプルコード

```python

# 悪い例 1
def create_NG_1(d: dict):
    # method内部でdictの値を設定する
    # 複雑な値を定義すればするほど見通しが悪くなる
    d['a'] = 1

    def _inner(d: dict):
        d['b'] = 2

    # 内部でさらにnestしているとさらに見通しが悪くなる
    _inner(d)


def main_NG():
    result = {}
    create_NG_1(result)

###################################################
# 改善案
def create_OK_1()　-> int:
    # keyに対するvalueを返す
    return 1

def create_OK_2() -> dict:
    # valueがdictならdictを返す

    def _inner() -> dict:
        # nestする場合は浅い階層のdictを返すmethodを定義
        return { 'e' : 2 }

    return {
        'c' : 1,
        'd' : _inner()
    }

def main_OK():
    # OK ver
    # valueは対応するmethodが返す
    # 少なくともmain_OKの範囲では、keyの存在の担保はできる
    result = {
        'a' : create_OK_1(),
        'b' : create_OK_2()
    }
```

## `__init__.py`に対し、不必要に import を書いたりロジックを書く

基本的に、`__init__.py`は空で問題ない。
本当に`__init__.py`に import を書いたりロジックを書く必要があるのか、立ち止まって考えること。

### 発生しうる問題

以下問題が発生する。

- 可読性が低くなる
- テストしづらい
- `method`の上書きなども発生する可能性がある
- 不要にクラスが import されうるため、スコープが分かり辛い(≒ グローバル変数と変わらない)

### 対策案

`__init__.py`に対し、本当に中身を書く必要があるのか考える。
個人的方針としては、PyPI の様にインストールして使う前提で**なければ**`__init__.py`は空で良い。

### バグサンプルコード

そもそも`import *`するな、という話もあるが、サンプルということで。
(`import *`するな、も本記事で書く予定)

#### フォルダ構成

```bash
├── main.py
├── sample
│   ├── __init__.py
│   └── sub_2.py
└── sub.py
```

#### コード

エントリーポイントは`main.py`

```python:sample/__init__.py
# __init__.pyに書く意味合いとしては、深い階層のフォルダに配置されているモジュールのAlias定義(import文の短縮)
from .sub_2 import sample_func
```

```python:sample/sub_2.py
# 同じ命名のmethodを定義
def sample_func():
    print('in sub_2')
```

```python:sub.py
# 同じ命名のmethodを定義
def sample_func():
    print('in sub')
```

```python:main.py
from sub import *
from sample import *

if __name__ == '__main__':
    print('main')
    sample_func()       # moduleのimportの順によって、callされるmethodが異なる(methodが上書きされる
```

## `import *`を使って、一律で method / class を import する

なにより lint を使え。`flake8`でも、なんでも良い。`flake8`なら F403 で怒られるはず。対応しろ。

### 対策案

`import` するとき、method や class は明示する。

### バグサンプルコード

```python
# 悪い例
from datetime import *

# 良い例
from datetime import datetime, date

# 改行ver
from datetime import (
    datetime,
    date
)
```

### その他

import が多くなる、という意見もあるだろうが、そういう状況が起こっている場合は、責務多重でクラス分割がうまくいっていないと思われる。
適切なクラス設計、分割、もしくはファイル分割をして import を減らす方向にすることで保守性が上がるはず。その方向で検討すべき。
