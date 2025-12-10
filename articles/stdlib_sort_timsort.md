---
title: "python の sort の正体は何なのか？"
emoji: "🐺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python3, algorithm, sort, timsort]
published: false
---

> アドベントカレンダー（自称）vol.10

皆さん、こんにちは。yumyum116 です。SWE転職を目指す者です。
アドベントカレンダーに便乗して、1日1記事投稿に挑戦するとともに、執筆活動の習慣化にも挑戦してみます。

この記事は、`プログラミング言語の標準ライブラリの仕様が気になって調べてみた`シリーズです。
プログラミング初学者の方の参考になれば嬉しいです。

今回は、python で使用可能なソート関数 sort について取り上げます。

## 1. sort の正体は何なのか
python では、配列の値をソートすることを考えた時に、手軽に利用できる関数として、sort があります。

LeetCode など、解答に速さが求められる場合には自作ソート関数ではなく、標準ライブラリとして使用可能な関数を使うことになると思います。
私も、LeetCode のコンテストに参加した時に、配列をソートするために sort を利用したことがあります。しかし、驚いたことに、なんとTLEになったのです。

標準ライブラリであるならば、時間計算量や空間計算量など、総合的に判断してベストなアルゴリズムが実装されているものと信じて疑っていなかったので、TLEという文字を見ることになるとは思っていませんでした。

この時の経験がきっかけとなり、本記事を執筆するに至りました。

-----

さて、sort 関数のアルゴリズムは何でしょうか。

挿入ソートでしょうか、選択ソートでしょうか、マージソートでしょうか、あるいはクイックソートでしょうか。


実は、上記に挙げたアルゴリズムのどれでもなく、また、sort 関数のアルゴリズムはpython のバージョンにより異なります。

バージョン2.3からバージョン3.10までは `Timsort`、バージョン3.11以降は、`Powersort` と呼ばれるアルゴリズムにより、実装されています。

本記事では、`Timsort` について取り上げます。


話が逸れますが、python には、利用可能なソート関数として sort と sorted があります。両者の違いについて、ここで簡単に触れておきます。

端的に説明すると、両者の違いは関数を適用できるオブジェクトと、戻り値の有無です。

sort はリストオブジェクトに対して利用可能な関数で、　戻り値がありません。
一方で、sorted は反復可能なオブジェクト（iterable : list, taple, dictionary,string, ...etc） に対して利用することができ、ソートされたオブジェクトを返します。

それぞれ、以下のイメージです。

```
list = [1, 4, 2, 5]


# sort()
list.sort()
print(list) # Output : [1, 2, 4, 5]

# sorted()
new_list = sorted(list)
print(list) # Output : [1, 4, 2, 5]
print(new_list) # Output : [1, 2, 4, 5]

```

しかしながら、戻り値の有無の違いはあるものの、実装されているアルゴリズムは同じです。


さて、話を戻します。

`Timsort` とは、挿入ソートとマージソートを組合せたソートアルゴリズムで、Tim Peter によって2002年に開発されました。

本記事では、python のソートアルゴリズムとして実装されているアルゴリズムとしてご紹介しますが、Java SE7 や Android も標準ソートアルゴリズムとして `Timsort` を採用しているようです。[出典](https://leapcell.io/blog/ja/python-no-soto-ga-omou-yori-hayai-riyu)


`Timsort` は高速、かつ安定性の高いアルゴリズムであると言われていて、平均時間計算量、最悪時間計算量ともに $O(nlogn)$ です。
挿入ソートやクイックソートの最悪時間計算量が $O(n^2)$ なので、これらのソートアルゴリズムと比較すると、場合によっては速いアルゴリズムと言えるでしょう。

さて、次章で Timsort の中身を読み解きます。


## 2. Timsort のアルゴリズムを読み解く

Timsort を C言語で実装したオリジナルコードは[こちら](https://svn.python.org/projects/python/trunk/Objects/listobject.c)にあります。
オリジナルコードは非常に複雑、かつ　C言語で書かれているので、本記事では Timsort の概念を説明しつつ、より簡易的な実装をベースに説明したいと思います。


### Timsort の概念
Timsort は、粗い説明をすると、次のような手順を踏みます。

　①ソートする対象の配列を、いくつかのソート済のかたまり "run" にまとめる。run にまとめる際には、挿入ソートを用いる。
　②①でまとめた run を、マージソートを使ってマージする。

Timsort の概念については、Gaurav Sen さんの[動画](https://www.youtube.com/watch?v=emeME__917E)にて分かりやすく説明されています。
インド訛りの英語ですが、インド訛りの英語の中では格段に聞きやすいかと思います。

動画は2パートに分かれていて、Part 1 で概念の説明を、Part 2 でアルゴリズムの挙動の説明をしています。
動画を視聴する上で、一つ注意が必要で、彼の動画の中では、run を chunk と表現しています。chunk という新しい概念が登場するわけではありません。
尺も15分程度ですので、手軽に見れます。お時間のある時にどうぞ。


さて、説明はほどほどにして、具体的にどのような実装になるのかを見てみます。
[Code is Art](https://www.youtube.com/watch?v=9yOV_03qlS8)の動画で実装されているコードが分かり良いと思います。

```js:tim_sort.py
def tim_sort(array):
	# Set the minimum run length
	min_run = 32															･･･(i)
	n = len(array)

	# Sort individual subarrays of size min_run using insertion sort
	for i in range(0, n, min_run):
		insertion_sort(array, i, min((i + min_run - 1), n - 1))

	# Double the size of the current run until the entire array is sorted
	size = min_run
	while size < n:
		for start in range(0, n, size * 2):									･･･(ii)
			mid = start + size - 1
			end = min((start + size * 2 - 1), (n - 1))
			merge(array, start, mid, end)
		size *= 2

# Helper function to perform insertion sort on subarrays
def insertion_sort(array, left, right):
	for i in range(left + 1, right + 1):
		key_item = array[i]
		j = i - 1
		while j >= left and array[j] > key_item:
			array[j + 1] = array[k]
			j -= 1
		array[j + 1] = key_item

# Helper function to merge subarrays
def merge(array, start, mid, end):											･･･(iii)
	if mid == end:
		return
	merged_array = []
	left_idx = start
	right_idx = mid + 1
	while left_idx <= mid and right_idx <= end:
		if array[left_idx] < array[right_idx]:
			merged_array.append(array[left_idx])
			left_idx += 1
		else:
			merged_array.append(array[right_idx])
			right_idx += 1
	while left_idx <= mid:
		merged_array.append(array[left_idx])
		left_idx += 1
	while right_idx <= end:
		merged_array.append(array[right_idx])
		right_idx += 1
	for i, sorted_item in enumerate(merged_array):
		array[start + i] = sorted_item

```

プログラムにおいて、採番した箇所について少し説明します。

(i)
ここで、min_run の値を32とハードコーディングしている理由は、挿入ソートが最速となる要素数の最小が32であることが知られているからです。上限は64です。
したがって、max_run の値を設定する場合には、max_run = 64 とするとよいと思います。

(ii)
run をマージする際に、コピー用の配列のサイズを min_run の値を用いて確保しています。
これは、min_run の値を用いるというより、**マージする run のうち、要素数の少ない方の run をコピーする**と理解しておくとよいと思います。

その理由は、メモリの確保領域を小さくすることで、マージソートの実行を速くするためです。
[別記事](https://zenn.dev/yumyum116/articles/lang_overhead)で、オーバーヘッドについてご紹介していますが、メモリ確保にかかる時間もオーバーヘッドに含まれます。
このオーバーヘッドをできるだけ少なくするために、小さい方のrunを複製するアプローチをとっています。

(iii)
ここでは、メモリの隣り合う領域の run 同士をマージしています。
隣り合う領域をマージすることには、とくに Timsort においては次のような効力があります。

*メモリの前の領域にある配列において、後ろの配列の先頭よりも小さい要素からなる部分配列が存在する場合には、その部分配列は並び順を変える必要がない
*メモリの後ろの領域にある配列において、前の配列の末尾の要素よりも大きい要素からなる部分配列が存在する場合には、その部分配列は並び順を変えることなくマージできる

これが成立する可能性がある理由は、run 自体がすでにソート済の配列であり、かつ、各 run はお互いにメモリが隣り合うという性質があるためです。

このメモリについても、Timsort の概念において外せない要点があるのですが、後日加筆修正するか、あるいは記事を分けて説明することにします。

## 3. 要素の並びがランダムな配列に対しては、遅くなる可能性もある
Timsort も常に万能であるわけではありません。
場合によっては、比較的高速に実行できますが、どのような配列であっても期待通りに実行できるわけではありません。
[公式のドキュメント](https://svn.python.org/projects/python/trunk/Objects/listsort.txt)にも記載があるように、要素の並びがランダムな配列に対しては、実行時間が期待値よりも遅くなることがあるようです。

Timsort を適用する配列の特性に応じて、どのアルゴリズムが最適であるのかを判断することも重要です。

※公式のドキュメントの詳細な内容については、後日加筆修正いたします。

-----

本記事では、Timsort について説明してみました。
初学者でも読める記事を目指したため、内容の厳密さは低いです。
自分の理解が浅いところもあるため、もう少し時間をかけて Timsort を理解した上で、記事の再執筆または加筆修正するかたちで内容をアップデートしたいと思います。

また、本記事にてご紹介した Timsort のプログラムは、一例に過ぎません。初学者でも理解できる内容のものを採用しました。
他にも、Timsortを実装されている方はいらっしゃるので、探してみてください。


記事内に誤謬等ある場合には、修正いたします。その際は、ご連絡いただけますと幸いです。

それでは、また。
