---
title: "print 関数の呼び出しは負荷がかかるよ、という話"
emoji: "🐺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python3, print, built-in]
published: false
---

> アドベントカレンダー（自称）vol.12

皆さん、こんにちは。yumyum116 です。SWE転職を目指す者です。
アドベントカレンダーに便乗して、1日1記事投稿に挑戦するとともに、執筆活動の習慣化にも挑戦してみます。

この記事は、`プログラミング言語の標準ライブラリの仕様が気になって調べてみた`シリーズです。
プログラミング初学者の方の参考になれば嬉しいです。

勉強がてら問題を解いていた時に、**時間計算量が少ないアルゴリズムを使用したにも関わらず、TLEになった**ことがあります。
原因を紐解いていくと、結果を出力する際に **print関数を呼び出す回数が多すぎた**ことが原因だったことが判明しました。

そこで、今回は、python の組み込み関数である print 関数について取り上げ、**なぜ print 関数の呼び出し回数が多いと TLE になるのか** について迫ります。


## 1. アルゴリズムの時間計算量は最適化されていても、TLEになる例
以下に、**アルゴリズムの時間計算量は最適化されていたものの、TLE になったプログラム**をご紹介します。
`エラトステネスのふるい`を用いて、素数判定を行うプログラムです。どこが原因で TLE になったか、想像がつくでしょうか？

補足すると、標準入力として渡される値は、次の条件を満たします。

> 条件
> $ 1 <= n <= 380,000$
> $ 1 <= array[i] <= 6,000,000 (1 <= i <= n) $


```js:etatosthenes_tle.py
#! /usr/bin/env python

MAX_A = 6000000

def eratosthenes(n):
    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False

    for i in range(2, int(n ** 0.5) + 1):
        if is_prime[i]:
            for j in range(i * i, n + 1, i):
                is_prime[j] = False

    return is_prime

n = int(input())
arr = [int(input()) for _ in range(n)]

for i in range(n):
    print("prime" if is_prime(arr[i]) else "not prime")
```

次に、TLE を解消したプログラムをご紹介します。

```js:eratosthenes.py
#! /usr/bin/env python

MAX_A = 6000000

def eratosthenes(n):
    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False

    for i in range(2, int(n ** 0.5) + 1):
        if is_prime[i]:
            for j in range(i * i, n + 1, i):
                is_prime[j] = False

    return is_prime

n = int(input())
arr = [int(input()) for _ in range(n)]

is_prime_table = eratosthenes(MAX_A)

out = []
for x in arr:
    out.append("prime" if is_prime_table[x] else "not prime")

print("\n".join(out))
```

さて、次章で TLE の原因を紐解いていきます。

## 2. python のプログラムを実行すると、何が起こっているのか
本題に移る前に、python プログラムを実行した時の処理の流れを説明します。
長くなりますが、お付き合いください。

1. python プログラム（.py）を実行する
---python インタプリタ---
2. 字句解析（プログラムをトークンに分解する処理）を行う
3. トークンを生成する
4. 構文解析を実行する
5. AST（抽象構文木）を生成する
6. コードオブジェクトを生成する
7. バイトコードを生成する
8. 仮想マシンがバイトコードを実行する
---python インタプリタ---
9. CPU が機械語を実行する

具体例を用いて、説明してみます。

例えば、python で次のプログラムを実行してみることにします（1）。

```js:test.py
print("Hi, how are you?")
```

### 字句解析
上記のプログラム test.py が python インタプリタに渡されると、python インタプリタはまず字句解析を行います。
字句解析を実行すると、プログラムは「Hi」「,」「how」「are」「you」「?」というトークンに分解されます。

### 構文解析
字句解析によって得られたトークンから、構文解析によって `AST（Abstract Syntax Tree : 抽象構文木）` と呼ばれるデータ構造を生成します。
実際に、test.py を実行した場合に生成されるASTオブジェクトを確認してみましょう。

```
$ python
> import ast
> tree = ast.parse('print("Hi, how are you?")')
>
> print(ast.dump(tree, indent=4))
Module(
    body=[
        Expr(
            value=Call(
                func=Name(id='print', ctx=Load()),
                args=[
                    Constant(value='Hi, how are you?')],
                keywords=[]))],
    type_ignores=[])
>

```

より "構造" を実感したい場合には、算術演算子を含む文字列を渡すとより理解が深まりますが、本記事では割愛します。

さて、出力された AST を読み解きます。

① `func=Name(id ='print', ctx=Load())`
これは、識別子 `print` を、「値として読み込む（Load）」という意味です。
Name の引数 ctx は、変数の格納 `Store()`、読み込み `Load()`、削除 `Del()` を渡すことができます。

構造的に表現すると、この一行は次のように表現できます。

> Name ノードは次の２つの情報を持つ構文ノードである
1. 識別子名 print がコード中に出現したという事実
2. 識別子名がどのような文脈で使われているのか（例の場合は、構文上 "読み込み" の役割として使われている）

② `args=[Constant(value='Hi, how are you?')]`
ここでは、func オブジェクトに対して渡す引数の値を表す構文ノードを示しています。CS的に表現するのであれば、**文字列リテラルを表す ASTノード** です。
ちなみに、`Constant` は Python 3.8 以降から使われていて、Python 3.7 以前では `Str` が使われていました。

③ `Call(...)`
関数呼び出し構文としての記述であり、次の３つの情報を保持します。

>i. `func` : 呼び出す対象の情報
ii. `args` : 引数として評価される式
iii. `keywords` : キーワード引数として評価される式

④ `Expr(...)`
出力のみを目的とした式であることを表現しています。
他にもノード（Expr と同じ階層のノード）は役割に応じて色々ありますが、紙幅の都合により、また別の記事でご紹介します。

⑤ `Module(...)`
`.py` ファイル1ファイル分の AST であることを示します。
補足ですが、body=[...]は構文要素の中に含まれる文（stmt）のリストであり、type_ignores=[] は型チェッカーのための補助情報です。型無視コメントの行番号などが格納されます。

### コードオブジェクトを生成する
次の２つの処理を実行します。

> ①ASTを元にして、次の処理を行う。

(i) 変数が local / global / free のどれであるかを決定する
(ii) 定数を定数テーブルに登録する
(iii) 関数・クラスごとに code object を作る

> ② `PyCodeObject`構造体を構築する

概念としては、次のような構造体が構築されます。

```
CodeObject {
    co_consts
    co_names
    co_varnames
    co_freevars
    co_cellvars
    co_flags
    co_code         **この時点ではまだ空 or 未確定**
}
```


### バイトコードを生成する
構文解析実行後、python インタプリタは AST からバイトコードを生成します。
バイトコードを生成する際は、`compile.c`というプログラムファイルが実行されます。これは、ASTからバイトコードを生成するコンパイラです。

バイトコードは、次のように表現されます。

```
>> import dis
>>> dis.dis('print("Hi, how are you?")')
  0           0 RESUME                   0

  1           2 PUSH_NULL
              4 LOAD_NAME                0 (print)
              6 LOAD_CONST               0 ('Hi, how are you?')
              8 CALL                     1
             16 RETURN_VALUE

```

これは、アセンブリ言語や機械語のプログラムに性質が近く、「LOAD_NAME 0」がバイトコードの命令を表します。
バイトコードの命令の種類は、`opcode.h` から確認することができます。

CS的に表現すると、**ASTを走査しながら、スタックマシン用の命令に変換する**処理です。

イメージとしては、次のような命令が生成されます。

```
co_code = [
    LOAD_NAME print
    LOAD_CONST "Hi, how are you?"
    CALL 1
    RETURN_VALUE
]

```
この処理によって、VMが直接解釈できるかたちになります。

### 仮想マシンがバイトコードを実行する
「バイトコードを生成する」で少し触れていますが、Python の仮想マシンは、演算で主に「スタック」を使用する「スタックマシン」と呼ばれるタイプのものです。
スタックは、積読のように、データをどんどん積み重ねていき、一番上にあるデータから取り出す性質を持ちます。この性質は LIFO (Last In First Out)と呼ばれます。

Python インタプリタが生成するバイトコードは、スタックマシンの仮想マシンで実行しやすいかたちになっています。
先ほどご紹介したバイトコードを説明すると、次のようになります。説明に簡略化のため、厳密さはやや落としています。

| 命令                                  | 意味                                                                    |
| ------------------------------------ | ----------------------------------------------------------------------  |
| RESUME        0                      | 関数の始まりを表す                                                         |
| PUSH_NULL                            | NULLをスタックに格納する（メソッド呼び出しではないことを示す印）                  |
| LOAD_NAME     0 (print)              | 変数 print の値をスタックに格納する                                          |
| LOAD_CONST    0 ('Hi, how are you?') | 変数 'Hi, how are you?' の値をスタックに格納する                             |
| CALL          1                      | スタックの上から、argc で指定された個数分の引数を取得し、呼び出し可能な関数を呼び出す |
| RETURN_VALUE                         | 呼び出し元に戻る                                                           |

仮想マシンでは、このバイトコードを上から順に実行し、生成された Python オブジェクトをスタックに格納する処理を実行しています。

### CPU が機械語を実行する
CPUはメモリにコピーされたプログラムを実行します。Python の場合には、python の仮想マシンの機械語を実行しています。

機械語について簡単に説明します。

機械語では、0 と 1 のみを組合せた２進数の値を使って、命令を表現します。
人間が理解できる形式で記述する場合には、16進数が使われます。興味がある方は、`.pyc`ファイルをバイナリエディタで開いてみるとよいと思います。

test.py の場合は次のようになります。（※人間が読むことのできる16進数で表現されており、CPUが実際に実行する機械語とは異なります）

![](images/tle_io/machinery_lang.png)

話を戻します。

例えば、バイトコードの `CALL 1` は、実際には次のように表される「Cの switch 文の case」を呼び出す命令です。

```
switch (opcode){
    case CALL:
    /* 引数を取り出して関数呼び出し */
}
```
仮想マシンから CPU の実行までの流れを正確に表現するならば、

> Cpython の C実装において、`CALL 1` の命令コードを読み取り、
switch(opcode) の `case CALL:` に分岐し、
CPUにおいては、その `case CALL:` に対応する CPython 自体の機械語を実行する

となります。

呼び出す関数としては、python レイヤでは print と記述していますが、呼び出される実体はCPython のC実装であり、`builtin_print_impl` に該当します。

ここまでが、python プログラムを実行した際に行われる処理の流れです。


## 3. print 関数では、何が起きているのか
さて、話を戻して print 関数の挙動について考えてみます。

print関数について簡単に説明すると、python の標準ライブラリではなく、組み込み関数です。
組み込み関数は、CPython 本体の Cソースコードで定義されます。print 関数の実体は [こちら](https://github.com/python/cpython/blob/v3.11.2/Python/bltinmodule.c#L1986)から確認できます。print 関数に該当する部分は、前述の通り `builtin_print_impl`です。

話を分かりやすくするため、中身を以下に貼付します。

```js:builtinmodule.c
static PyObject *
builtin_print_impl(PyObject *module, PyObject *args, PyObject *sep,
                   PyObject *end, PyObject *file, int flush)
/*[clinic end generated code: output=3cfc0940f5bc237b input=c143c575d24fe665]*/
{
    int i, err;

    if (file == Py_None) {
        PyThreadState *tstate = _PyThreadState_GET();
        file = _PySys_GetAttr(tstate, &_Py_ID(stdout));
        if (file == NULL) {
            PyErr_SetString(PyExc_RuntimeError, "lost sys.stdout");
            return NULL;
        }

        /* sys.stdout may be None when FILE* stdout isn't connected */
        if (file == Py_None) {
            Py_RETURN_NONE;
        }
    }

    if (sep == Py_None) {
        sep = NULL;
    }
    else if (sep && !PyUnicode_Check(sep)) {
        PyErr_Format(PyExc_TypeError,
                     "sep must be None or a string, not %.200s",
                     Py_TYPE(sep)->tp_name);
        return NULL;
    }
    if (end == Py_None) {
        end = NULL;
    }
    else if (end && !PyUnicode_Check(end)) {
        PyErr_Format(PyExc_TypeError,
                     "end must be None or a string, not %.200s",
                     Py_TYPE(end)->tp_name);
        return NULL;
    }

    for (i = 0; i < PyTuple_GET_SIZE(args); i++) {
        if (i > 0) {
            if (sep == NULL) {
                err = PyFile_WriteString(" ", file);
            }
            else {
                err = PyFile_WriteObject(sep, file, Py_PRINT_RAW);
            }
            if (err) {
                return NULL;
            }
        }
        err = PyFile_WriteObject(PyTuple_GET_ITEM(args, i), file, Py_PRINT_RAW);
        if (err) {
            return NULL;
        }
    }

    if (end == NULL) {
        err = PyFile_WriteString("\n", file);
    }
    else {
        err = PyFile_WriteObject(end, file, Py_PRINT_RAW);
    }
    if (err) {
        return NULL;
    }

    if (flush) {
        PyObject *tmp = PyObject_CallMethodNoArgs(file, &_Py_ID(flush));
        if (tmp == NULL) {
            return NULL;
        }
        Py_DECREF(tmp);
    }

    Py_RETURN_NONE;
}
```

print 関数においては、次の流れで標準出力に文字が表示されます。

1. Python コードを実行する
2. Python バイトコードを生成する
---ここまでが Python レイヤ ---
3. CPython の VM を実行する
4. builtin print を呼び出す
5. file.write() を呼び出す
6. libc write() を呼び出す
7. OS カーネル syscall を呼び出す
8. 標準出力に出力する

前章との繋がりが読み取りにくいため、各処理の説明に移る前に、少し補足します。
上記の流れは、表の動きを追った表現であり、CPU はずっと稼働しています。

CPU視点で書き直すと

>CPython の仮想マシンを構成する C の機械語を
CPU が実行している最中に、
CALL 命令に対応する処理として builtin print の機械語が呼ばれ、
その処理の中で PyFile_WriteObject → FileIO.write → write syscall
へと **CPU の実行対象が遷移していく**

イメージです。

図示すると次のように表されます。

```
CPU
 └─ CPython VM（機械語）
      └─ builtin print（機械語）
           └─ PyFile_WriteObject（機械語）
                └─ FileIO.write（機械語）
                     └─ libc write（機械語）
                          └─ kernel write（機械語）
```

さて、ここまで整理できたところで、いよいよ print 関数の処理の説明に移ります。
1, 2, 3 は前章で触れた部分のため割愛します。ここでは、4以降の流れを追っていきます。

### builtin print を呼び出す
再三の繰り返しになりますが、実体としては

```
static PyObject *
builtin_print_impl(PyObject *module, PyObject *args, PyObject *sep,
                   PyObject *end, PyObject *file, int flush)
```
が呼ばれます。

この時に実行されることは、次の３つです。

> ①引数（PyObject*）を受け取る
②`sep`, `end`, `file`, `flush` を解釈する
③出力先 `file` を決定する（デフォルトは `sys.stdout`）

### file.write() を呼び出す
次のC実装により、python の file オブジェクトの write メソッドを呼び出しています。

```
PyFile_WriteObject(obj, file, Py_PRINT_RAW);
```

上記の `builtinmodule.c` の中では、次の部分が該当します。

```
PyFile_WriteObject(sep, file, Py_PRINT_RAW)
```
`sys.stdout.write(...)` と同等です。

### libc write を呼び出す
python の `sys.stdout` は、次のような多段ラップ構造になっています。

```
TextIOWrapper
  └─ BufferedWriter
       └─ FileIO
```

FileIO.write() の C実装は、`Module/_io/fileio.c` に記述されており、`_io_FileIO_write_impl` が該当する部分です。

```js:cpython/Modules/_io/fileio.c
/*[clinic input]
_io.FileIO.write
    cls: defining_class
    b: Py_buffer
    /

Write buffer b to file, return number of bytes written.

Only makes one system call, so not all of the data may be written.
The number of bytes actually written is returned.  In non-blocking mode,
returns None if the write would block.
[clinic start generated code]*/

static PyObject *
_io_FileIO_write_impl(fileio *self, PyTypeObject *cls, Py_buffer *b)
/*[clinic end generated code: output=927e25be80f3b77b input=2776314f043088f5]*/
{
    Py_ssize_t n;
    int err;

    if (self->fd < 0)
        return err_closed();
    if (!self->writable) {
        _PyIO_State *state = get_io_state_by_cls(cls);
        return err_mode(state, "writing");
    }

    n = _Py_write(self->fd, b->buf, b->len);
    /* copy errno because PyBuffer_Release() can indirectly modify it */
    err = errno;

    if (n < 0) {
        if (err == EAGAIN) {
            PyErr_Clear();
            Py_RETURN_NONE;
        }
        return NULL;
    }

    return PyLong_FromSsize_t(n);
}
```
（[出典](https://github.com/python/cpython/blob/main/Modules/_io/fileio.c)）

プロトタイプとしては、次のように表されます。

```
_Py_write(fd, buf, size)
```

この時点で、Python レイヤを抜けて C の I/O レイヤに入ります。

### OS カーネル syscall を呼び出す
print 関数の中では、ここが最も重い処理です。
この syscall では、次の３つの処理を実行します。

> ①ユーザ空間からカーネル空間へ委譲する
②コンテキストスイッチを行う
③OSが stdout を処理する

#### ユーザ空間からカーネル空間へ委譲する
これは、CPUの実行モードが切り替わることです。

**ユーザ空間**とは、普通のアプリ（Python, CPython, libc）が動く場所であり、**デバイスやメモリには触れません。**
**カーネル空間**とは、OS本体が動く場所で、デバイス操作、ファイル書き込み、プロセス管理などを実行することができます。

print 関数は、デバイス操作を要求するため、ユーザ空間からカーネル空間に処理を委譲する必要があります。
syscall(write)を呼ぶと発生すると捉えていただけるとよいと思います。

#### コンテキストスイッチを行う
CPUが「実行状態」を切り替えることです。

コンテキストスイッチには、前述した (A) ユーザモードからカーネルモードへの切り替え と、(B) プロセス切り替え の2種類がありますが、print関数において重要なコンテキストスイッチは(A) です。

syscall によるモード切替が、I/O が遅い最大の理由の一つです。

#### OS が stdout を処理する
簡潔に書くと、`「fd=1 に書きなさい」と言われた内容を、実際の出力先に届ける処理`です。

stdout を処理するとは、概念的に表すと次のようになります。

```
sys_write(fd=1, buf)
  ↓
ファイル構造体を探す
  ↓
対応するデバイス or ファイルへ転送
  ↓
（必要ならバッファリング）

```

この一連の流れを、端的に表現するならば

> print() によって CPython が write() を呼び、
CPU が syscall によりユーザモードからカーネルモードへ切り替わり、
OS が fd=1（stdout）の実体を解決して、
端末・ファイル・パイプなどにデータを書き込む
となります。

このカーネル処理やデバイス I/O は、syscall のモード切替よりもさらに高コストです。

### 標準出力に出力する
カーネルから、実際の出力先（この場合は端末）に文字が届きます。

## 4. TLE の理由は、for 文の中で毎回呼び出される print 関数だった
まずはじめに、ここまでお読みいただいているあなたに、感謝の意を表したいと思います。
ありがとうございます。

そう、TLEの原因は見出しの通りです。

具体的には、冒頭でご紹介したプログラムの次の実装が、TLEの引き金となっています。

```
for i in range(n):
    print("prime" if is_prime(arr[i]) else "not prime")
```

このプログラムでは、インクリメント変数 $i$ が１増えるたびに print 関数が実行される設計になっています。
ここまでご説明してきた、プログラムの実行の流れに照らして考えると、毎回カーネル処理とデバイス I/O が実行されていることになります。

仮に、入力として渡される値の最大値である 380,000 が渡されたとすると、38万回 print 関数が呼び出されることになります。
CPUの気持ちになると、拷問でもされているような気分になることでしょう。
入力条件に対するアプローチが間違っているいい実例です。

一方で、修正後のプログラムを見てみましょう。

```
out = []
for x in arr:
    out.append("prime" if is_prime_table[x] else "not prime")

print("\n".join(out))
```

入力値がどれだけ大きい値であっても、出力する値を配列に格納することで print 関数の呼び出しを1回に抑えることができています。
38万回の呼び出しと、1回の呼び出しでは、CPUにかかる負荷が段違いに違うことは想像に難くないと思います。


**アルゴリズムを最適化しているにも関わらず、TLEになる場合には、I/Oの処理を疑ってみる** という新たな気づきと学びを教えてくれた失敗でした。

本記事を最後までお読みいただいたあなたも、重要な知識を身につけられたことと思います。

-----

記事内に誤謬等ございます場合には、修正いたします。その際は、ご連絡いただけますと幸いです。

それでは、また。

