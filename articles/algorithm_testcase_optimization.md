---
title: "テストケースは実装しているアルゴリズムの特性も加味して決める"
emoji: "🐺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python3, time complexity]
published: true
---

> アドベントカレンダー（自称）vol.8

皆さん、こんにちは。yumyum116 です。SWE転職を目指す者です。
アドベントカレンダーに便乗して、1日1記事投稿に挑戦するとともに、執筆活動の習慣化にも挑戦してみます。

本記事は、頭休めになるような、それでいて、学びにもなるtipsを取り扱います。
コーヒー片手にお付き合いいただけますと幸いです。

-----

さて、先日執筆した「アルゴリズムの裏に潜む大学数学 vol.1 : 素数を判定する」の記事の中で、異なるアルゴリズム間の実行時間の違いについて調べてみました。

https://zenn.dev/yumyum116/articles/math_uni_isprime

記事の中では、

> $n = 99999937$ とした時の実行時間を比較してみました。

と、さっくり書いていますが、この値にたどり着くまでの小話を本記事では綴りたいと思います。

-----

## 1. エッジケースの実行が、一瞬で終わってしまった

どのような値を渡せば、有意に実行時間に差が出るのか分からなかったため、テスト値は chatGPT に出してもらいました。

chatGPT が初めに提案した値は40桁の値で、時間計算量が O(n) となるプログラムであれば、実行に最大数十秒かかる一方で、他のプログラムであれば、最大数秒で終わるだろう、ということでした。

最大数十秒、というところにやや不安を感じながらも、40桁の値をテストケースとして渡してみたところ、何とも不思議な現象が起きました。


**何と、実行に最も時間がかかると思われたプログラムが、一瞬で計算を終えてしまった**

のです。


ちなみに、「実行に最も時間がかかる思われたプログラム」とは、次のプログラムです。

```js:is_prime_n.py
def is_prime_n(n):
	if n <= 1:
		return False

	for i in range(2, n):
		if n % i == 0:
			return False

	return True
```


おかげさまで、効率化を図った他のプログラムと、実行時間に差が出ない結果となりました。

もちろん、chatGPTにはハルシネーションのリスクもありますし、出力される内容は何らかの事実に基づくものではなく、出力される確率が一番高くなる単語を並べているだけですから、chatGPTが間違えている可能性もありますし、その一方で、"何か" が起こった可能性もあります。

現状、python のエキスパートでもない私は、ググる力もなく、嘘を教えたかもしれないchatGPTに何が起こっているのかを聞くほか、対策がありませんでした。


そして、chatGPTから返ってきた回答は、

**ループが回っていない可能性がある**

とのことでした。

その原因をたどってみると、実行に一番時間がかかると思われていたプログラムは、理論上は時間計算量が $O(n^{40})$ になると見積もられ、仮に計算が実行されたとしても、**永遠に終わらない** プログラムでした。

少し考えれば分かることですが、chatGPTが提案してきた40桁という数字に対して、「計算終わらないじゃん！」というツッコミを入れられなかった自分の理解と感度の低さも関係しているので、「chatGPTが嘘をついた！」とは言えません。

chatGPTに対して何かを質問する場合には、自分自身も相応に詳しい必要があることを痛感した、いい経験になったと捉えましょう。


一方で、プログラムが一瞬で終わったように見える謎については、chatGPTが諸説提案していますが、どの説が一番確からしいかは、また後日検証してみます。

この小話における教訓は、**テストケースを作成する際は、自分が書いたプログラムの時間計算量や空間計算量がどの程度になるのかを理解した上で、現実的に実行可能な値（エッジケース含む）を設定しよう**ということです。


## 2. おまけ（40桁で試したいなら）
今回は、実装するアルゴリズムの違いによる実行時間の違いに着目していたため、残念ながら40桁の値はテストケースに選ばれませんでした。

しかし、40桁に対しても十分に現実的な時間で、素数判定を完了できるアルゴリズムはあるようです。
それが、`Miller-Rabin法` と `Pollard's Rho法` です。後者は競プロの世界で好まれるアルゴリズムみたいですね。

実行時間の比較はまた後日実施してみますが、実装は次のようになります。

```js:miller_rabin.py
# Miller–Rabin primality test
def is_prime_mr(n):
    if n < 2:
        return False
    
    small_primes = [2, 3, 5, 7, 11, 13, 17, 19, 23]
    if n in small_primes:
        return True
    for p in small_primes:
        if n % p == 0:
            return False

    # n-1 = d * 2^s を満たす s, d を求める
    d = n - 1
    s = 0
    while d % 2 == 0:
        s += 1
        d //= 2

    # Miller-Rabin テスト本体
    def check(a):
        x = pow(a, d, n)
        if x == 1 or x == n - 1:
            return True
        for _ in range(s - 1):
            x = pow(x, 2, n)
            if x == n - 1:
                return True
        return False

    # 確定判定用の基数
    test_bases = [2, 325, 9375, 28178, 450775, 9780504, 1795265022]

    for a in test_bases:
        if a % n == 0:
            return True
        if not check(a):
            return False

    return True
```

```js:pollards_rho.py
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

そもそも、`Miller-Rabin法`や`Pollard's Rho法`が何なのか、という話もありますが、話が難しくなるので、本記事では取り上げません。
こんなアプローチもあるよ、ということでご紹介させていただきました。

-----

記事内に誤謬等ございましたら、修正いたします。
その際は、ご連絡いただけますと幸いです。

それでは、また。
