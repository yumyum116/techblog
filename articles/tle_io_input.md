---
title: "I/Oの最適化を考える〜入力処理の最適化〜"
emoji: "🐺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python3, input]
published: false
---

> アドベントカレンダー（自称）vol.24

皆さん、こんにちは。yumyum116 です。SWE転職を目指す者です。
アドベントカレンダーに便乗して、1日1記事投稿に挑戦するとともに、執筆活動の習慣化にも挑戦してみます。

この記事は、`プログラミング言語の標準ライブラリの仕様が気になって調べてみた`シリーズです。
プログラミング初学者の方の参考になれば嬉しいです。

本記事では、TLE をテーマに、input 関数を取り上げます。
先日、print 関数を繰り返し構文の中で呼び出すことが TLE の原因の一つになる、という記事を公開しました。

https://zenn.dev/yumyum116/articles/tle_io_print

こちらの記事では、アルゴリズムの計算量ベースでは TLE にならないことが想定されたにも関わらず、結果としては TLE となったため、原因を調べた結果、for 文の中で print 関数を呼び出していたことが原因である、という趣旨で書いています。

しかし、異なる観点で検討してみると、input 関数も処理としては重い関数であることが分かったため、本記事では `I/Oの最適化` というテーマで、input 関数について取り上げます。

## 1. input 関数は何をする関数なのか
input 関数は次のように使われます。

```
str = input()
```

ソースコードは [bltinmodule.c](https://github.com/python/cpython/blob/d246a6766b9d8cc625112906299c4cb019944300/Python/bltinmodule.c#L2236-L2237) の `builtin_input_impl(PyObject *module, PyObject *prompt)` をご覧ください。

関数の挙動としては、標準入力から1行分読み取るだけなのですが、実際には次のような処理が内部で実行されています。

1. 標準入力から1行読み取る(bytes)
```
s = PyOS_Readline(stdin, stdout, promptstr);
```

2. 末尾の改行文字を削除する
```
len = strlen(s);
(...)
else {
         (...)
        else {
            len--;  # 末尾の \n を削除
            if (len != 0 && s[len-1] == '\r')
                len--;
            result = PyUnicode_Decode(s, len, stdin_encoding_str, stdin_errors_str);
        }
}
```

3. Unicode 文字列としてデコードする(bytes -> str)
```
PyObject *result;
result = PyUnicode_Decode(s, len, stdin_encoding_str, stdin_errors_str);

/* 補足 */
stdin_encoding_str = PyUnicode_AsUTF8(stdin_encoding); # encoding 名を取り出す処理
```

4. Python の str オブジェクトを返す
もう少し正確に表現すると、`PyUnicode_Decode` の戻り値の型は、C API 上 `PyObject *` に統一されていますが、成功時には `Unicode(str)` を生成して返します。
Cpython においては、`Unicode Object` として取り扱われますが、とくに、python3 レイヤでは、`str` オブジェクトとして取り扱われます。

## 2. input 関数の内部構造
input() の内部処理は、次の通り、環境によって分岐します。

```
[対話入力]
OS
 ↓
readline
 ↓
PyOS_Readline
 ↓
input()

[非対話入力]
OS
 ↓
kernel buffer
 ↓
fd
 ↓
TextIOWrapper（sys.stdin）
 ↓
input()
```
対話環境では、`PyOS_Readline` を通じて readline が入力を処理し、ファイルやパイプ入力では、`TextIOWrapper` を通じて OS のファイルディスクリプタから読み込まれます。

## 3. なぜ input 関数はループで重くなるのか
前章でも述べたとおり、input 関数を一度実行するたびに、以下の処理が発生します。

1. Python レベルでの関数呼び出し
2. OSから読み込んだバイト列を元にした、文字列オブジェクトの生成
3. 改行除去（.rstrip('\n') 相当）
4. 指定されたエンコーディングに基づく、Unicode デコード処理

これらの処理の多くは、**入力サイズに依存しない固定コスト** であるため、I/O におけるボトルネックは、計算量（$O(N)$）ではなく、`1回あたりの定数時間` に支配されます。

したがって、input関数をループ内で何度も呼び出すと、軽い処理を大量に呼び出すことによる累積により、CPU計算よりも I/O 処理が支配的になります。

CPU 上の計算処理は、ループ回数が増えた場合であっても、キャッシュやパイプラインによって効率的に実行されます。
一方で input() は OS やランタイムとの境界を跨ぐ処理であり、毎回バッファ管理やオブジェクト生成を伴います。
この「境界越え」のコストは最適化されにくく、ループ回数に比例してそのまま累積します。

## 4. input() よりも処理が速い関数
`sys.stdin.readline()` や `sys.stdin.buffer.read()` は、input() よりも高速に動作します。
これは特別な最適化が施されているからではなく、input() が内部で行っている処理の一部を省略していることによるものです。

### sys.stdin.readline()
1 行分の文字列をそのまま返す低レベル API です。
input() と異なり、改行除去などの整形処理を行わないため、その分だけ処理が軽くなります。

ただし、readline() は常に最速というわけではありません。
呼び出し回数が多い場合、readline() もやはり関数呼び出しコストを持つため、根本的な解決にはなりません。

### sys.stdin.buffer.read()
input() や readline() とは発想が異なり、**「1 行ずつ読む」API ではなく、「入力全体を一度に読み込む」API** です。
buffer.read() は OS から受け取ったバイト列をそのまま返します。
そのため、Unicode デコードや改行処理といった高レベル処理を一切行いません。

input() がループで遅くなる原因は、「1 回あたりの定数コスト」が積み重なる点にありました。
buffer.read() はこの問題を、呼び出し回数を 1 回にまとめることで根本的に回避します。

これらの違いは、設計思想の違いとして整理できます。すなわち、

input() は安全で扱いやすい高レベル API
readline() は整形を省いた中レベル API
buffer.read() は速度を最優先した低レベル API

です。

高速入力とは「速い関数を使うこと」ではなく、必要な抽象度を選ぶことだと言えます。

使い分けの指針としては、

+ 通常のスクリプト：input()
+ 行数が多い競プロ：sys.stdin.readline()
+ 超大量入力：sys.stdin.buffer.read()

となるでしょう。

## 5. アンチパターン
### (1) とりあえず readline() に置き換える
前章で説明した通り、input() と readline() の違いは、改行処理などの整形処理を関数内で行うか、関数外で行うか、の違いであり、根本的な構造は同じです。
したがって、呼び出し回数が多い限り、定数倍は積み上がります。

### (2) TLE の原因が入力か未確認のまま、I/O最適化を検討する
TLE となった際に疑うべき優先度としては、

+ 計算量
+ データ構造
+ 無駄なループ

です。I/O は最後に疑うべきボトルネックであり、最初に改善すべき部分ではありません。

### (3) split() のコストを無視する
次のような処理が盲点になりやすいです。

```
line = sys.stdin.readline()
a, b, c = map(int, line.split())
```

処理としては、新しい `list` を作成し、各トークンを `str` として確保した後に、`int()` による変換を行っています。
I/O よりも Python オブジェクト生成が支配的になるケースが多いです。

仮に、高速入力にしたつもりであっても、その直後で同じだけのオブジェクトを生成するプログラムになっている場合、高速入力による恩恵は相殺されています。

### (4) 低レベル API を常用する
例えば、プログラムの高速入力として `sys.stdin.buffer.read()` を使うことは、可読性が低く、バグをい生みやすいため、実務レベルでは実用的ではありません。


高速入力は「速い関数を選ぶこと」ではなく、「どこがボトルネックかを見極めること」から始まります。
抽象化を剥がす前に、まずは本当に剥がす必要があるのかを考えることが重要です。

## 7. どのような時に I/O 最適化すべきか
I/O 最適化は強力ですが、常に必要なわけではありません。
重要なのは「速い関数を知っていること」ではなく、最適化すべき状況を見極めることです。

### (1) 入力サイズ・行数が明らかに多い時
例えば、次のような場合は、I/O の最適化を検討してもよいでしょう。

+ 入力行数が $10^5$ 〜 $10^6$ 行以上の時
+ 1 行あたりの処理は軽い（単なる数値読み取りなど）時

このような場合、計算量が $O(N)$ であっても、input() の 1 回あたりの定数コストが支配的になります。

### (2) 計算量を落としても TLE する場合
例えば、次のような場合が該当します。

+ アルゴリズムの時間計算量は $O(N)$ または $O(N log N)$
+ 無駄なループや重い処理はない
+ それでも実行時間制限に引っかかる

この場合、計算ではなく「読み書き」がボトルネックになっている可能性があります。

### (3) 入力処理がループの支配項になっている時
例えば、次のような構造です。

```
for _ in range(N):
    a, b = map(int, input().split())
```

ループの中心に I/O がある場合、入力関数の定数コストがそのまま全体性能に直結します。

### (4) 競技プログラミングなど、制限が厳しい文脈の時
競技プログラミングでは、実行時間制限が厳しいため、高速入力を使う選択が必然になります。


このように、**入力回数が多く、計算が軽いときに限り、I/O 最適化は効果を発揮します。**

## 8. input() は悪者ではない
ここまで、input() が重い理由を述べてきましたが、決して「遅いから使ってはいけない関数」ではありません。
安全で分かりやすく、ほとんどの用途では十分に高速な高レベル APIです。

input() が問題になるのは、次のような限られた状況だけであることをご理解いただければと思います。

+ 入力回数が非常に多い
+ 1 回あたりの処理が軽く、I/O がループの支配項になっている
+ 実行時間制限が厳しい（競技プログラミングなど）

このような場合に限って、`sys.stdin.readline()` や `sys.stdin.buffer.read()` といった より低レベルな入力手法が意味を持ちます。

### 参考
input() の C実装を以下に貼付します。

```
static PyObject *
builtin_input_impl(PyObject *module, PyObject *prompt)
/*[clinic end generated code: output=83db5a191e7a0d60 input=5e8bb70c2908fe3c]*/
{
    PyObject *fin = _PySys_GetObjectId(&PyId_stdin);
    PyObject *fout = _PySys_GetObjectId(&PyId_stdout);
    PyObject *ferr = _PySys_GetObjectId(&PyId_stderr);
    PyObject *tmp;
    long fd;
    int tty;

    /* Check that stdin/out/err are intact */
    if (fin == NULL || fin == Py_None) {
        PyErr_SetString(PyExc_RuntimeError,
                        "input(): lost sys.stdin");
        return NULL;
    }
    if (fout == NULL || fout == Py_None) {
        PyErr_SetString(PyExc_RuntimeError,
                        "input(): lost sys.stdout");
        return NULL;
    }
    if (ferr == NULL || ferr == Py_None) {
        PyErr_SetString(PyExc_RuntimeError,
                        "input(): lost sys.stderr");
        return NULL;
    }

    /* First of all, flush stderr */
    tmp = _PyObject_CallMethodId(ferr, &PyId_flush, NULL);
    if (tmp == NULL)
        PyErr_Clear();
    else
        Py_DECREF(tmp);

    /* We should only use (GNU) readline if Python's sys.stdin and
       sys.stdout are the same as C's stdin and stdout, because we
       need to pass it those. */
    tmp = _PyObject_CallMethodId(fin, &PyId_fileno, NULL);
    if (tmp == NULL) {
        PyErr_Clear();
        tty = 0;
    }
    else {
        fd = PyLong_AsLong(tmp);
        Py_DECREF(tmp);
        if (fd < 0 && PyErr_Occurred())
            return NULL;
        tty = fd == fileno(stdin) && isatty(fd);
    }
    if (tty) {
        tmp = _PyObject_CallMethodId(fout, &PyId_fileno, NULL);
        if (tmp == NULL) {
            PyErr_Clear();
            tty = 0;
        }
        else {
            fd = PyLong_AsLong(tmp);
            Py_DECREF(tmp);
            if (fd < 0 && PyErr_Occurred())
                return NULL;
            tty = fd == fileno(stdout) && isatty(fd);
        }
    }

    /* If we're interactive, use (GNU) readline */
    if (tty) {
        PyObject *po = NULL;
        const char *promptstr;
        char *s = NULL;
        PyObject *stdin_encoding = NULL, *stdin_errors = NULL;
        PyObject *stdout_encoding = NULL, *stdout_errors = NULL;
        const char *stdin_encoding_str, *stdin_errors_str;
        PyObject *result;
        size_t len;

        /* stdin is a text stream, so it must have an encoding. */
        stdin_encoding = _PyObject_GetAttrId(fin, &PyId_encoding);
        stdin_errors = _PyObject_GetAttrId(fin, &PyId_errors);
        if (!stdin_encoding || !stdin_errors ||
                !PyUnicode_Check(stdin_encoding) ||
                !PyUnicode_Check(stdin_errors)) {
            tty = 0;
            goto _readline_errors;
        }
        stdin_encoding_str = PyUnicode_AsUTF8(stdin_encoding);
        stdin_errors_str = PyUnicode_AsUTF8(stdin_errors);
        if (!stdin_encoding_str || !stdin_errors_str)
            goto _readline_errors;
        tmp = _PyObject_CallMethodId(fout, &PyId_flush, NULL);
        if (tmp == NULL)
            PyErr_Clear();
        else
            Py_DECREF(tmp);
        if (prompt != NULL) {
            /* We have a prompt, encode it as stdout would */
            const char *stdout_encoding_str, *stdout_errors_str;
            PyObject *stringpo;
            stdout_encoding = _PyObject_GetAttrId(fout, &PyId_encoding);
            stdout_errors = _PyObject_GetAttrId(fout, &PyId_errors);
            if (!stdout_encoding || !stdout_errors ||
                    !PyUnicode_Check(stdout_encoding) ||
                    !PyUnicode_Check(stdout_errors)) {
                tty = 0;
                goto _readline_errors;
            }
            stdout_encoding_str = PyUnicode_AsUTF8(stdout_encoding);
            stdout_errors_str = PyUnicode_AsUTF8(stdout_errors);
            if (!stdout_encoding_str || !stdout_errors_str)
                goto _readline_errors;
            stringpo = PyObject_Str(prompt);
            if (stringpo == NULL)
                goto _readline_errors;
            po = PyUnicode_AsEncodedString(stringpo,
                stdout_encoding_str, stdout_errors_str);
            Py_CLEAR(stdout_encoding);
            Py_CLEAR(stdout_errors);
            Py_CLEAR(stringpo);
            if (po == NULL)
                goto _readline_errors;
            assert(PyBytes_Check(po));
            promptstr = PyBytes_AS_STRING(po);
        }
        else {
            po = NULL;
            promptstr = "";
        }
        s = PyOS_Readline(stdin, stdout, promptstr);
        if (s == NULL) {
            PyErr_CheckSignals();
            if (!PyErr_Occurred())
                PyErr_SetNone(PyExc_KeyboardInterrupt);
            goto _readline_errors;
        }

        len = strlen(s);
        if (len == 0) {
            PyErr_SetNone(PyExc_EOFError);
            result = NULL;
        }
        else {
            if (len > PY_SSIZE_T_MAX) {
                PyErr_SetString(PyExc_OverflowError,
                                "input: input too long");
                result = NULL;
            }
            else {
                len--;   /* strip trailing '\n' */
                if (len != 0 && s[len-1] == '\r')
                    len--;   /* strip trailing '\r' */
                result = PyUnicode_Decode(s, len, stdin_encoding_str,
                                                  stdin_errors_str);
            }
        }
        Py_DECREF(stdin_encoding);
        Py_DECREF(stdin_errors);
        Py_XDECREF(po);
        PyMem_FREE(s);
        return result;

    _readline_errors:
        Py_XDECREF(stdin_encoding);
        Py_XDECREF(stdout_encoding);
        Py_XDECREF(stdin_errors);
        Py_XDECREF(stdout_errors);
        Py_XDECREF(po);
        if (tty)
            return NULL;

        PyErr_Clear();
    }

    /* Fallback if we're not interactive */
    if (prompt != NULL) {
        if (PyFile_WriteObject(prompt, fout, Py_PRINT_RAW) != 0)
            return NULL;
    }
    tmp = _PyObject_CallMethodId(fout, &PyId_flush, NULL);
    if (tmp == NULL)
        PyErr_Clear();
    else
        Py_DECREF(tmp);
    return PyFile_GetLine(fin, -1);
}
```

-----

記事内に誤謬等ございます場合には、修正いたします。その際はご連絡いただけますと幸いです。

それでは、また。
