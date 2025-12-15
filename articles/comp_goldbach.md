---
title: "繰り返し処理の実行範囲を最適化して、剰余演算を行わない設計にするだけで、プログラムの実行時間が125倍速くなった話"
emoji: "🐺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python3, algorithm, goldbach, ゴールドバッハ予想]
published: false
---

> アドベントカレンダー（自称）vol.15

皆さん、こんにちは。yumyum116 です。SWE転職を目指す者です。
アドベントカレンダーに便乗して、1日1記事投稿に挑戦するとともに、執筆活動の習慣化にも挑戦してみます。

本記事は、頭休めになるような、それでいて、学びにもなるtipsを取り扱います。
コーヒー片手にお付き合いいただけますと幸いです。

先日、「プログラムを書き始める前に、数式でどのように表されるのかを考える〜ゴールドバッハ予想を例に〜」というタイトルで、自分が当初考えたアルゴリズムと、課題要件を数式で表し、より効率的なアプローチで解くアルゴリズムをそれぞれご紹介しました。

https://zenn.dev/yumyum116/articles/math_algebra_goldbach

本記事では、実際に実行時間にどの程度差が出るのかを検証してみます。

## 1. アルゴリズムの振り返り
まずは、それぞれのアルゴリズムを再掲します。

### goldbach_original.py

1. 素数リストを作成する
2. 1 で作成した素数リストを用いて、２つの素数の和が $N$ となる素数の組を線形探索により探索する
3. 2 で見つけた素数の組合せを、配列に格納する。
4. 3 で作成した配列の中から、積が最大となる組合せのインデックスを調べる
5. 4 で調べたインデックスの位置にある配列の要素を出力する

```js:goldbach_original.py
def goldbach_conjecture(n):
    prime = [2]
    for i in range(3, n + 1, 2):
        for j in range(3, i):
            if i % j == 0:
                break
        else:
            prime.append(i)

    prime_combi = []
    for i in range(len(prime)):
        if n - prime[i] in prime:
            prime_combi.append([prime[i], n - prime[i]])
    if not prime_combi:
            return None

    max_res = -1
    max_idx = 0
    for j in range(len(prime_combi)):
        prod = prime_combi[j][0] * prime_combi[j][1]
        if prod > max_res:      // 判定式で値の更新有無を確認しておらず、積が最大となるか判定できないため、修正しました。
            max_res = prod
            max_idx = j

    return prime_combi[max_idx]

n = int(input())
result = goldbach_conjecture(n)
if result is None:
	print("No combination found")
else:
	for prime in result:
		print(prime)
```

### goldbach.py

1. 素数リストを作成する
2. 1 で作成した素数リストを用いて、２つの素数の和が $N$ となる素数の組を線形探索により探索する。なお、探索は $\frac{N}{2}$ から始めるものとする。
3. 2 の探索により見つかった素数の組を戻り値として返す

```js:goldbach.py
def goldbach_conjecture(n):
    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False

    for i in range(2, int(n ** 0.5) + 1):
        if is_prime[i]:
            for j in range(i * i, n + 1, i):
                is_prime[j] = False

    for p in range(n // 2, 1, -1):
        q = n - p
        if is_prime[p] and is_prime[q]:
            return p, q

n = int(input())
p, q = goldbach_conjecture(n)
print(p)
print(q)
```

## 2. 実行時間の差を検証してみる
それぞれのアルゴリズムの実行時間の差が有意に出るように、入力データとして与える値を現実的な範囲で大きい値にしてみます。
今回は、$n = 34240$ としました。

実行結果はこちらです。

```
===== Running original version =====

--- goldbach_original.py ---
17117
17123

real    0m2.779s
user    0m2.756s
sys     0m0.005s

===== Running optimized version =====

--- goldbach.py ---
17117 17123

real    0m0.017s
user    0m0.012s
sys     0m0.005s

===== ALL DONE =====

```

主に、ユーザレベルのCPU使用率で大きく差があることが分かります。
これは、アルゴリズムの計算量差が如実に出たことを表しますが、具体的にどのレベルで実行に時間がかかったのかを、分解してみましょう。

```
===== Running original version =====

$ perf stat python goldbach_org.py

17117
17123

 Performance counter stats for 'python goldbach_org.py':

           2742.45 msec task-clock                       #    0.991 CPUs utilized
               202      context-switches                 #   73.657 /sec
                12      cpu-migrations                   #    4.376 /sec
               974      page-faults                      #  355.156 /sec

       2.767778938 seconds time elapsed

       2.732186000 seconds user
       0.009978000 seconds sys

==============   DONE   ==============

===== Running optimized version =====

$ perf stat python goldbach.py
17117 17123

 Performance counter stats for 'python goldbach.py':

             21.81 msec task-clock                       #    0.806 CPUs utilized
                49      context-switches                 #    2.246 K/sec
                 1      cpu-migrations                   #   45.846 /sec
               989      page-faults                      #   45.342 K/sec

       0.027048899 seconds time elapsed

       0.018157000 seconds user
       0.004034000 seconds sys

==============   DONE   ==============

```

`task-clock` の違いが大きな原因であることが分かりました。この指標は、**CPUをどれだけ長時間占有しているか**を表す指標です。

`goldbach_org.py` では、**１つのCPUを占有し続けており、CPUが計算でフル稼働している**結果となりました。`goldbach.py`と比較すると、約125倍以上の差です。
`context-switches` や `page-faults` でも差が認められるものの、`task-clock` の差と比較すると無視できるほど十分に小さいため、無視して問題ないでしょう。


続いて、具体的にどの処理で実行に時間がかかっているのかを、深堀りしてみます。
行単位での実行時間を確認するため、`line_profiler` を使ってみます。

```
===== Running original version =====

Timer unit: 1e-06 s

Total time: 52.1322 s
File: goldbach_org.py
Function: goldbach_conjecture at line 3

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
     3                                           @profile
     4                                           def goldbach_conjecture(n):
     5         1          1.1      1.1      0.0      prime = [2]
     6     17120       7741.8      0.5      0.0      for i in range(3, n + 1, 2):
     7  59073721   24941173.2      0.4     47.8          for j in range(3, i):
     8  59070065   27071607.1      0.5     51.9              if i % j == 0:
     9     13463       5951.6      0.4      0.0                  break
    10                                                   else:
    11      3656       2320.7      0.6      0.0              prime.append(i)
    12
    13         1          1.2      1.2      0.0      prime_combi = []
    14      3658       1496.9      0.4      0.0      for i in range(len(prime)):
    15      3657     100186.0     27.4      0.2          if n - prime[i] in prime:
    16       702        554.7      0.8      0.0              prime_combi.append([prime[i], n - prime[i]])
    17         1          1.1      1.1      0.0      if not prime_combi:
    18                                                       return None
    19
    20         1          0.8      0.8      0.0      max_res = -1
    21         1          0.6      0.6      0.0      max_idx = 0
    22       703        278.1      0.4      0.0      for j in range(len(prime_combi)):
    23       702        324.8      0.5      0.0          prod = prime_combi[j][0] * prime_combi[j][1]
    24       702        278.0      0.4      0.0          if prod > max_res:
    25       351        136.4      0.4      0.0              max_res = prod
    26       351        135.0      0.4      0.0              max_idx = j
    27
    28         1          3.3      3.3      0.0      return prime_combi[max_idx]

==============   DONE   ==============

===== Running optimized version =====

Timer unit: 1e-06 s

Total time: 0.0527341 s
File: goldbach.py
Function: goldbach_conjecture at line 3

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
     3                                           @profile
     4                                           def goldbach_conjecture(n):
     5         1        159.1    159.1      0.3      is_prime = [True] * (n + 1)
     6         1          1.1      1.1      0.0      is_prime[0] = is_prime[1] = False
     7
     8       185         90.0      0.5      0.2      for i in range(2, int(n ** 0.5) + 1):
     9       184         95.0      0.5      0.2          if is_prime[i]:
    10     62654      26714.0      0.4     50.7              for j in range(i * i, n + 1, i):
    11     62612      25656.8      0.4     48.7                  is_prime[j] = False
    12
    13         4          6.0      1.5      0.0      for p in range(n // 2, 1, -1):
    14         4          4.0      1.0      0.0          q = n - p
    15         4          4.3      1.1      0.0          if is_prime[p] and is_prime[q]:
    16         1          3.8      3.8      0.0              return p, q

==============   DONE   ==============
```

原因が明確に分かりましたね。素数判定の二重ループの実行時間が全体の99%を占める結果となりました。
実行時間が長くなる理由は、主に ①そもそも python における剰余の計算が重いこと、②本質的には不要な範囲まで繰り返し処理を実行していることです。

とくに、②については
```
for j in range(3, i):
```
を、$ \sqrt{i}$ までの範囲に限定することで、大幅に計算回数を削減することができます。
計算が $ \sqrt{i}$ までで十分であることは、`合成数 $a$ は、必ず $\sqrt{n}$ 以下の素因数 $b$ を持つ` という性質に基づくものです。

続いて、①については python に起因する理由です。
少しずつ分解して説明してみます。

```
if i % j == 0:
```

### 1. 「%」は Python では「C言語 の %」ではない

C言語の int 同士の % は「レジスタ上の整数」であるため、演算処理は比較的軽いです。
一方、Python の i と j は PyLong（任意精度整数）オブジェクトとして扱われます。
内部的には、

①型チェック
②オブジェクト参照
③関数ディスパッチ
④例外条件の分岐
⑤除算アルゴリズム呼び出し

を経由します。

### 2. 「%」の実体：CPython の処理フロー

`if i % j == 0` は、概念的には次のステップを実行します。

① i と j をスタックから取り出す
② i % j を計算する
	(i) PyNumber_Remainder(i, j) のような“汎用演算”を呼ぶ
	(ii) ここで i と j が int（PyLong）だと分かり
	(iii) long_mod / l_mod のような PyLong 専用ルーチンに進む
③ 剰余の結果（PyLongオブジェクト）を生成する
④ ③の結果が 0（PyLong）と等しいか比較する
⑤ if 判定

このうち「②〜④」の処理が重いです。

### 3. “任意精度”のコスト：結果が新しいオブジェクトとして生成される

C言語であれば、i % j の結果はレジスタに乗るだけですが、Python は i % j の結果を 新しい PyLong オブジェクトとして生成する可能性が高く、そのために、`計算結果として生成されたオブジェクトを 0 (数値ではなく、オブジェクト)と比較する` という **オブジェクト世界の手続き** が実行されます。

### 4. 除算は CPU 的に遅い部類

剰余演算は内部的に「除算」を含みます。乗算・加算はパイプラインに乗りやすい一方で、除算・剰余は一般にレイテンシが大きい性質があります。

Python では、この性質に加えて、演算の前後に型・参照・分岐処理が乗るため、高コストになる、という構造があります。


以上の理由から、**実行が重くなるべくして重くなるプログラム設計である**ということが分かりました。

-----

記事内に誤謬等ある場合は、修正いたします。その際は、ご連絡いただけますと幸いです。

それでは、また。
