---
title: "エラトステネスのふるいを高速化してみる"
emoji: "🐺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python3, algorithm, eratosthenes, エラトステネスのふるい]
published: false
---

> アドベントカレンダー（自称）vol.22

皆さん、こんにちは。yumyum116 です。SWE転職を目指す者です。
アドベントカレンダーに便乗して、1日1記事投稿に挑戦するとともに、執筆活動の習慣化にも挑戦してみます。

本記事は、頭休めになるような、それでいて、学びにもなるtipsを取り扱います。
コーヒー片手にお付き合いいただけますと幸いです。

「print関数の呼び出しは負荷がかかるよ、という話」という記事の中で、効率的なアルゴリズムとして「エラトステネスのふるい」をご紹介していますが、本記事では、この「エラトステネスのふるい」をさらに高速化することを考えてみます。

https://zenn.dev/yumyum116/articles/tle_io_print

## 1. アルゴリズムの振り返り
まずは、エラトステネスのふるいを再掲します。

```js:eratosthenes.py
def eratosthenes(n):
    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False

    for i in range(2, int(n ** 0.5) + 1):
        if is_prime[i]:
            for j in range(i * i, n + 1, i):
                is_prime[j] = False

    return is_prime
```

時間計算量は $O(NloglogN)$ です。本記事では、**時間計算量を変えずに、実装レベルでどこまで高速化できるか** を検証してみます。

## 2. 最適化案①偶数を処理しない
次の発想に基づき、偶数の処理を skip するアルゴリズムを考えてみます。

+ 2以外の偶数はすべて合成数
+ 奇数だけ管理することで、配列サイズおよびループ回数を約1/2に抑えられる

実装は次のようになります。

```js:eratosthenes_evenskip.py
def eratosthenes_odd(n):
    if n < 2:
        return []

    size = (n + 1) // 2
    is_prime = [True] * size
    is_prime[0] = False

    limit = int(n ** 0.5) // 2
    for i in range(1, limit + 1):
        if is_prime[i]:
            p = 2 * i + 1
            start = (p * p) // 2
            for j in range(start, size, p):
                is_prime[j] = False

    return is_prime
```

## 3. 最適化案② list[bool] を使わず、bytearray を使う
次の発想に基づき、素数リストを list[bool] ではなく、bytearray で作成することを考えてみます。

+ `bool` は Python オブジェクト
+ `bytearray` は 1 byte の連続メモリである
+ CPU キャッシュに優しい

実装は次のようになります。

```js:eratosthenes_bytearray.py
def eratosthenes_bytearray(n):
    is_prime = bytearray(b"\x01") * (n + 1)
    is_prime[0:2] = b"\x00\x00"

    for i in range(2, int(n ** 0.5) + 1):
        if is_prime[i]:
            for j in range(i * i, n + 1, i):
                is_prime[j] = 0

    return is_prime
```

## 4. 最適化案③ 最適化案①と②のハイブリッド
実装は次のようになります。

```js:eratosthenes_hybrid.py
def eratosthenes_fast(n):
    if n < 2:
        return []

    size = (n + 1) // 2
    is_prime = bytearray(b"\x01") * size
    is_prime[0] = 0

    limit = int(n ** 0.5) // 2
    for i in range(1, limit + 1):
        if is_prime[i]:
            p = 2 * i + 1
            start = (p * p) // 2
            is_prime[start::p] = b"\x00" * (((size - start - 1) // p) + 1)

    return is_prime
```

## 5. 実行時間の違いを比較してみた
入力値を $10^7$ として、実行時間にどの程度差がでるのかを検証してみました。
なお、検証環境では `time` が bash の組み込みキーワードであり実行ファイルとして存在しないことから、最大メモリが取得できなかったため、実行時間のみを計測しています。

```
Benchmark n=10000000
==========================

>>> eratosthenes.py

real    0m0.634s
user    0m0.604s
sys     0m0.017s

>>> eratosthenes_odd.py

real    0m0.245s
user    0m0.237s
sys     0m0.007s

>>> eratosthenes_bytearray.py

real    0m0.801s
user    0m0.771s
sys     0m0.006s

>>> eratosthenes_fast.py

real    0m0.028s
user    0m0.018s
sys     0m0.010s
```

最適化案①の偶数処理をスキップする実装では、実行時間が約2.6倍向上する結果となりました。
最適化案②のデータ構造をbytearrayに置き換える実装では、実行時間が0.8倍となり、メモリ最適化が速度最適化とはならないことが確認できました。
最適化案③のハイブリッド実装では、実行時間が約22.6倍向上する結果となりました。

最適化案③では、次の実装で表される **スライス代入** を採用することにより、**forループを回さずにCレベルで一括処理を実行している** ことにより、高速化を実現しています。

```
is_prime[start::p] = b"\x00" * (((size - start - 1) // p) + 1)
```

以上の結果から、Python においては **ループ構造とメモリアクセスの設計が性能改善の要素として支配的である** ことが分かりました。

-----

本記事で取り上げた、スライス代入については、また記事を分けてご紹介したいと思います。
高速化テクニックとして、何か学びになる内容が執筆できていましたら幸いです。

記事内に誤謬等ございます場合には、修正いたします。その際は、ご連絡いただけますと幸いです。

それでは、また。
