---
title: "Pollard's-Rho 法を理解する"
emoji: "🐺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python3, algorithm, pollard-rho, ポラード・ロー法]
published: false
---

> アドベントカレンダー（自称）vol.19

皆さん、こんにちは。yumyum116 です。SWE転職を目指す者です。
アドベントカレンダーに便乗して、1日1記事投稿に挑戦するとともに、執筆活動の習慣化にも挑戦してみます。

先日、「素数判定を行うプログラムのアルゴリズム間の実行時間の違い」を調査するためのテストケース考案の裏話に関する記事を公開しました。

https://zenn.dev/yumyum116/articles/algorithm_testcase_optimization

実行時間差テストの中では、検証対象プログラムで実装しているアルゴリズムの時間計算量をベースに、8桁の数値をテストケースとして選定し、当初 chatGPT が提案した40桁の数値は、プログラムに実装しているアルゴリズムでは TLE になるという理由で、テストケースから除外しました。

しかし、テストケースの数値が40桁であっても、十分に現実的な時間でプログラムの実行が完了するアルゴリズムの一つとして、`Pollard's-Rho法` を紹介しています。

本記事では、この `Pollard's-Rho法` を取り上げます。

## 1. Pollard's-Rho法とは
自然数 $n$ が素数であるかどうかを判定し、素数であれば終了、合成数であれば $n$ の約数を探し、またその約数が素数であれば素因数として出力に追加、合成数であればさらにその数に対して再帰的に約数の探索を行うことを繰り返すアルゴリズムです。

Pollard's-Rho法では、$O(n^\frac{1}{4})$ で50%以上の確率で素因数を発見することができると言われていますが、理論的な保証はないヒューリスティックスです。

## 2. アルゴリズムの手順
合成数 $n$ について、次の手順により素因数分解を行います。

1. $n$ が合成数である間、ステップ2, 3 を繰り返す
2. $n$ の素因数 $p$ を求める
3. $n$ を $p$ で割り続け、割った回数分、$p$ を出力に追加する
4. $n$ が１より大きければ、$n$ は素数であるため、$n$ を出力に追加する

1 の素数判定には `Miller-Rabin法` を用います。
上記のステップを実装したものが、次のプログラムです。

```pollard's-rho.py
import random
import math

# Miller-Rabin
def is_prime_mr(n):
    if n < 2:
        return False
    small_primes = [2,3,5,7,11,13,17,19,23]
    if n in small_primes:
        return True
    for p in small_primes:
        if n % p == 0:
            return False

    d = n - 1
    s = 0
    while d % 2 == 0:
        s += 1
        d //= 2

    def check(a):
        x = pow(a, d, n)
        if x == 1 or x == n-1:
            return True
        for _ in range(s-1):
            x = pow(x, 2, n)
            if x == n-1:
                return True
        return False

    test_bases = [2, 325, 9375, 28178, 450775, 9780504, 1795265022]
    for a in test_bases:
        if a % n == 0:
            return True
        if not check(a):
            return False

    return True


# Pollard’s Rho
def pollards_rho(n):
    if n % 2 == 0:
        return 2
    if n % 3 == 0:
        return 3

    while True:
        x = random.randrange(2, n - 1)
        y = x
        c = random.randrange(1, n - 1)

        f = lambda v: (pow(v, 2, n) + c) % n

        d = 1
        while d == 1:
            x = f(x)
            y = f(f(y))
            d = math.gcd(abs(x - y), n)
        if d != n:
            return d


# n の素因数分解
def factor(n, res=None):
    if res is None:
        res = []
    if n == 1:
        return res
    if is_prime_mr(n):
        res.append(n)
    else:
        d = pollards_rho(n)
        factor(d, res)
        factor(n // d, res)
    return res
```

-----

記事内に誤謬等ございます場合には、修正いたします。その際は、ご連絡いただけますと幸いです。

それでは、また。
