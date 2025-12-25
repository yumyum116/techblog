---
title: "クイックソートの分割戦略を比べてみる"
emoji: "🐺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python3, algorithm, クイックソート, Hoare partition, Lomuto partition]
published: false
---

> アドベントカレンダー（自称）： vol.25

皆さん、こんにちは。yumyum116 です。SWE転職を目指す者です。
アドベントカレンダーに便乗して、1日1記事投稿に挑戦するとともに、執筆活動の習慣化にも挑戦してみます。

アドベントカレンダーとしては、ようやく最終記事になりました。。。！
アドベントカレンダーの最終記事としてお届けするテーマは `クイックソートの分割戦略の比較` です。

クイックソートに関する解説の多くにおいては、`Lomuto` が紹介されがちですが、分割戦略においては **実装効率、比較回数、スワップ回数** に明確な差があります。
本記事では、クイックソートの分割処理に焦点を当て、

+ Hoare partition
+ Lomuto partition

をアルゴリズム、実装、性能の観点から比較します。

## 1. クイックソートにおける「分割（Partition）」とは何か
クイックソートの全体像を簡潔に整理すると、次のようになります。

+ pivot を選ぶ
+ 配列を pivot を境に左右に分割する
+ 再帰的にソートする

ここで、分割処理が性能を左右する理由は、冒頭でも述べた通り、**比較回数、　swap回数、メモリアクセスパターン** に明確な差が生じるためです。
クイックソートの性能は、分割処理の効率により決まるといっても過言ではありません。

## 2. Lomuto partition の仕組み
### (1) アルゴリズム概要
pivot として選定した要素を基に、配列を分割するアルゴリズムです。
基本的には、配列の末尾の要素を pivot に選定し、1本の走査で pivot 未満の要素を左側へ、pivot 以上の要素を右側へ移動させます。
最終的に、pivot 自身を境界位置へ swap することで分割が完了します。

### (2) 実装例
Lomuto partition のアルゴリズム実装例は次のようになります。
あくまで一例と捉えていただければと思います。

```
def	partition(array):
	n = len(array)
	pivot = array[n - 1]

	i = -1 # pivot よりも小さい要素と大きい要素の境界として振る舞う
	for j in range(n - 1):
		if array[j] < pivot:
			i += 1
			array[i], array[j] = array[j], array[i]

	# pivot を確定位置へ swap させる
	array[i + 1], array[n - 1] = array[n - 1], array[i + 1]
```

### (3) アルゴリズムの特徴
Lomuto partition には次のような特徴があります。

+ 実装が直感的で分かりやすい
+ swap 回数が多くなりやすい

## 3. Hoare partition の仕組み
### (1) アルゴリズム概要
Lomuto partition 同様、pivot として選定した要素を基に、配列を分割するアルゴリズムです。
ただし、Lomuto partition とは異なり、pivot 自体を最終的な位置に固定しません。
その代わり、pivot を基準に「左側は pivot 以下、右側は pivot 以上」という境界を返します。

### (2) 実装例
Hoare partition のアルゴリズム実装例は次のようになります。

```
def partition(array):
	n = len(array)
	pivot = array[0]

	i = -1
	j = n    # 最初の j -= 1 により、　j = n - 1 となる。
	while True:
		while True:
			i += 1
			if array[i] >= pivot:
				break
		while True:
			j -= 1
			if array[j] <= pivot:
				break

		if i > j:
			break

		array[i], array[j] = array[j], array[i]

	return j
```

### (3) アルゴリズムの特徴
Hoare partition には次のような特徴があります。

+ swap 回数が少ない
+ partition の戻り値の扱い注意が必要

## 5. 実装上の違い
両者には、主に再帰区間の違いとバグを生みやすいポイントの点において違いがあります。

### (1) 再帰区間の違い
+ Lomuto
 + quick_sort(left, p-1)
 + quick_sort(p+1, right)
+ Hoare
 + quick_sort(left, p)
 + quick_sort(p+1, right)
※Hoare partition が返す p は pivot の位置ではなく、「分割境界」を表すインデックスであるため、このような再帰区間になります。

### (2) バグを生みやすいポイント
+ Hoare で無限ループになる典型例
+ pivot と等しい値が多い場合の挙動差

## 6. どちらの分割戦略を使うべきか？
ここまで、Lomuto partition と Hoare partition の仕組みと実装上の違いを見てきました。
では、実際にクイックソートを実装する際、どちらの分割戦略を選ぶべきなのでしょうか。

結論としては、「用途によって使い分ける」が最も現実的な答えになります。

以下に、それぞれの分割戦略が向いているケースを列挙します。

### (1) Lomuto partition が向いているケース
Lomuto partition は、次のようなケースに向いています。

+ クイックソートの仕組みを学習・説明したい場合
+ アルゴリズムの流れを直感的に理解したい場合
+ 実装の簡潔さ・読みやすさを重視したい場合

Lomuto partition は、「pivot 未満を左に集め、最後に pivot を確定位置へ移動させる」という流れが明快で、直感的に理解しやすい点が大きな特徴です。
一方で、swap 回数が増えやすく、性能面では不利になるケースがある点には注意が必要です。

### (2) Hoare partition が向いているケース
Hoare partition は、次のようなケースで有効です。

+ 実行速度や swap 回数を重視したい場合
+ 競技プログラミングや実務コードで用いる場合
+ 大規模な配列や性能がボトルネックになる場面

Hoare partition は、左右からポインタを動かして不要な swap を減らすため、Lomuto partition と比べて 実行効率が高くなることが多い という特徴があります。

その反面、pivot の最終位置が確定しない点や、再帰区間の取り扱いに注意が必要であり、実装を誤るとバグを生みやすい側面もあります。


どちらが優れているか、という観点ではなく、**何を重視するか** によって選ぶべき分割戦略が異なることをご理解いただけたかと思います。

-----

記事内に誤謬等ございます場合には、修正いたします。その際は、ご連絡いただけますと幸いです。

それでは、また。
