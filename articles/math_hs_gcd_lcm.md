---
title: "アルゴリズムの裏に潜む高校数学 ver.1 : 最大公約数と最小公倍数"
emoji: ":Wolf"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python3, algorithm, gcd, lcm]
published: true
---

> アドベントカレンダー（自称）vol.4

皆さん、こんにちは。yumyum116 です。SWE転職を目指す者です。
アドベントカレンダーに便乗して、1日1記事投稿に挑戦するとともに、執筆活動の習慣化にも挑戦してみます。

この記事は、`アルゴリズムの裏に潜む高校数学を紐解く`シリーズです。
アルゴリズムによって、アルゴリズムを支える数学が、高校数学だったり、大学以降で学習する内容だったり、大学数学の中でも大学３年生以上の専門分化以降で学習する内容だったりするので、数学のレベル別にトピックを分けてみます。
もちろん、自分が理解できる範囲の内容しか書けないので、記事の充足度に偏りが出ることは予めご承知おきください。

自分と同じようなところで躓いている方、プログラミング初学者の方の参考になれば嬉しいです。

シリーズ第１稿目は、最大公約数および最小公倍数を求めるアルゴリズムの裏に潜む数学について紐解きます。

## 1. 最大公約数・最小公倍数を求めるプログラム
さて、記事の主旨はアルゴリズムから概念を紐解くことなので、普段の記事とは逆の流れで説明します。

最大公約数（gcd : greatest common denominator）、最小公倍数（lcm : least common multiple）を求めるプログラムを以下に示します。
本記事では、まず数字を素因数分解した後に、因数と指数の値を用いて、最大公約数および最小公倍数を求めるアプローチをベースに考えます。
勘の良い方は、ここまで読んだだけでもう分かったかもしれません。

```js:gcd_lcm.py

# 整数を素因数分解する
def factorize(number):
    primes = {}

    for i in range(2, number + 1):
        if n % i > 0:
            continue

        exponential = 0
        while number % i == 0:
            exponential += 1
            number //= i

    if number != 1:
        primes[number] = 1

    return primes

# 最大公約数を求める
def calculate_gcd(a, b):
    factorize_a = factorize(a)
    factorize_b = factorize(b)
    factorize_gcd = {}

    for prime in factorize_a:
        exponential = factorize_a[prime]
        if prime in factorize_b:
            exponential = min(exponential, factorize_b[prime])
            factorize_gcd[prime] = exponential

    gcd = 1
    for factor in factorize_gcd:
        exponential = factorize_gcd[factor]
        for _ in range(exponential):
            gcd *= factor

    return gcd

# 最小公倍数を求める
def calculate_lcm(a, b):
    factorize_a = factorize(a)
    factorize_b = factorize(b)
    factorize_lcm = {}

    for prime in factorize_a:
        exponential = factorize_a[prime]
        if prime in factorize_b:
            exponential = max(exponential, factorize_b[prime])
            factorize_lcm[prime] = exponential

    lcm = 1
    for factor in factorize_lcm:
        exponential = factorize_lcm[factor]
        for _ in range(exponential):
            lcm *= factor

    return lcm

```

次の章で、このプログラムの裏に潜む数学について説明します。

## 2. どのような数学的知識が必要なのか？
前章で紹介したプログラムは、次に示す　`整数の性質`を利用しています。

> ２つ以上の整数について、
> ①共通する素因数の個数が最も小さいものを掛け合わせた数が最大公約数になる
> ②それぞれの因数の個数が最も大きいものを掛け合わせた数が最小公倍数になる

具体例を用いて考えてみます。

32 と 40 の最大公約数、最小公倍数を求めてみましょう。
素因数分解を活用して解くことを考えると、32, 40 はそれぞれ次のように素因数分解できます。

$32 = 2^5$
$40 = 2^3 * 5$

この結果を用いると、最大公約数および最小公倍数は、それぞれ次のようになります。

最大公約数：
　$2^3 = 8$

最小公倍数：
　$2^5 * 5 = 160$

## 3. 1 の別解
異なるアプローチとして、以下のようなアプローチも考えられます。

### (1) 最小公倍数と最大公約数の性質を用いる
最小公倍数と最大公約数の間には、次のような関係があります。

> 正の整数 $a$, $b$ に対して、それらの最大公約数を $g$, 最小公倍数を $l$ とおくと、 $ab = gl$
> つまり、「最大公約数と最小公倍数の積」が「もとの二つの数の積」に等しい。

この性質を利用するならば、一方の関数はより簡易的に書くことができます。

例えば、
```js:gcd_lcm_ver2.py
def calculate_gcd(a, b):
    ...
    return gcd

def calculate_lcm(a, b, gcd):
    gcd = calculate_gcd(a, b)

    lcm = (a * b) // gcd

    return lcm
```

さらに、高校数学の知識を使わずとも、小学校で学ぶ算数の知識で２つ以上の整数の最大公約数および最小公倍数を求めることもできます。
プログラムはわざわざ書きませんが、各整数を割り切ることができる素数で割り続け、これ以上割り切れないところまで計算したら、出てきた素数を掛け合わせたものが最大公約数、出てきた素数と、残った余りをすべて掛け合わせたものが最小公倍数、とする解き方です。

ただし、この解き方の場合は必要となる素数がどの値までなのかが事前に分からないため、現実解とは言えないでしょう。

-----

さて、本記事では `アルゴリズムから紐解く高校数学` をご紹介しました。
初学者の方や、大学等で専門的にプログラミングを学んでいない場合には、アルゴリズムを学ぶ過程で「なぜこのように書かれるのか分からない」部分が必ず出てくると思います。

その場合は、プログラムの裏に前提として必要な数学的知識が潜んでいることがほとんどです。
焦らずに、一つずつ紐解いていきましょう。

記事内に、誤謬等ございましたら修正いたします。その際は、ご連絡いただけますと幸いです。

それでは、また。
