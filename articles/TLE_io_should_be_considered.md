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


## 1. print 関数では、何が起きているのか
python の print 関数は標準ライブラリではなく、組み込み関数です。
組み込み関数は、CPython 本体の Cソースコードで定義されます。print 関数の実体は [こちら](https://github.com/python/cpython/blob/v3.11.2/Python/bltinmodule.c#L1986)から確認できます。print 関数に該当する部分は、`builtin_print_impl`です。

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

まず、python のコードが実行されてからの流れを説明します。

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

仮想マシンでは、このバイトコードを上から順に実行し、実行結果をスタックに格納する処理が行われています。

### CPU が機械語を実行する
CPUはメモリにコピーされたプログラムを実行します。このプログラムには、コンピュータに実行させたい命令が、「機械語」と呼ばれる形式で書かれています。
機械語では、0 と 1 のみを組合せた２進数の値を使って、命令を表現します。
人間が理解できる形式で記述する場合には、16進数が使われます。興味がある方は、`.pyc`ファイルをバイナリエディタで開いてみるとよいと思います。

