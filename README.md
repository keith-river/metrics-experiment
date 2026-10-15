# metrics-experiment
Experiment of Software Metrics 

# ソフトウェア評価 ガイダンス：簡易ハンズオン（約20分）

予備実験（0週目）の流れを、バブルソートを題材に、**手を動かして一周**する様子を駆け足で実演する。

ゴールは次の2つである。

- **「書く → 整形 → リント → テスト → 修正 → カバレッジ」**という点検の流れを、実際のコマンドと出力で体験する。

- ★⇒　説明の都合上、**料理番組形式＝あらかじめ仕込んでおいた材料を、テンポよく提示するために、git**を使用する
- git の詳しい説明はしない。
- 作業の区切りごとに `git commit` して、ツールが変えた部分を **`git diff` の before/after** で確認する
- 使うのは `init`、`switch -c`、`add`、`commit`、`diff`、`log` だけで、**ステップごとにブランチを切って**、各段階の成果物をリポジトリに残す

-----

## 1. 全体の流れと時間配分

|時間|ステップ|やること|ブランチ|
|-|-|-|-|
|1.5分|0|作業ディレクトリを作り、`git init`|`main`|
|2分|1|`粗削り版`を置いてコンパイル、コミット|`step1-draft`|
|2.5分|2|`clang-format` → `git diff` で before/after|`step2-format`|
|3.5分|3|`clang-tidy` の指摘を読み、直すものを決めて反映|`step3-lint`|
|2.5分|4|最初のテストを実行 → **失敗する**|`step4-test`|
|2分|5|原因を直す → `git diff` → テストが通る|`step5-fix`|
|4分|6|`gcov`／`lcov`：通っても **63.6%** → テストを足して100%|`step6-coverage`|
|2分|まとめ|ブランチ一覧と、今日の学びの確認|`main`|
|（予備）|7|ランダムテストとベンチマーク（時間があれば／各自で）|`step7-random-bench`|

合計で約20分。
各ステップの最後にコミットし、次のステップの最初に `git switch -c` で新しいブランチを切る。**つまずいたら、そのステップのブランチ名を `git switch` すれば、正解の状態に戻れる**（「5. 困ったとき」を参照）。

-----

## 2. 事前準備（配布物と環境）

```
git clone https://github.com/keith-river/metrics-experiment.git
cd metrics-experment
unzip se-handson-kit.zip
mv se-handson-kit ~/
cd se-handson-kit
```


配布物（`se-handson-kit/`）の中身：

|ファイル|内容|
|-|-|
|`step0/`、`step1/`、`step3/`〜`step7/`|各ステップで使うファイル（`cp` で作業ディレクトリに置く。ステップ2は `clang-format` を実行するだけなので、ファイルはない）|
|`env.sh`|`$CC`（コンパイラ）、`$GCOV`、`$LCOV_GCOV` を環境に合わせて決める|
|`check-env.sh`|ツールがそろっているかの点検|
|`demo-auto.sh`|ハンズオン全工程を自動再生（リハーサル・非常用）|
|`reference-repo/`|全ブランチが入った完成版のリポジトリ（見本）|
|`sample-logs/`|リハーサルで得た実行ログ（この資料に載せたものと同じ）|

事前に **各自で1回だけ** 次を実行してもらう：

```
. ./env.sh
bash ./check-env.sh
git config --global user.name  "あなたの名前"
git config --global user.email "you@example.com"
git config --list
```
- 一度でもgit を使っていれば，リポジトリ毎に`git config --global user...`する必要なし

```
  **Mac での注意**：標準の `gcc` は実体が Apple clang で、`gcov` が動かない。
   Homebrew で `brew install gcc lcov llvm clang-format` を入れると（`clang-tidy` は `llvm` に含まれる）、
   `env.sh` が `gcc-14` などを自動で選ぶ。`check-env.sh` が `OK カバレッジ計測` と出れば準備完了。
  Homebrew の llvm は PATH に入らないので、`export PATH="$(brew --prefix llvm)/bin:$PATH"` を `~/.zshrc` に追加する。
```

-----

## 3. シナリオ　@ 計算機室ＶＭ （Debian）

以降のコマンドは、すべて作業ディレクトリ `~/se-handson` で行う。出力は、リハーサルで実際に出たものである（`clang-tidy` の版が違うと、件数や文言が少し変わる）。

### ステップ0（1.5分）作業ディレクトリと git の準備

★⇒ **作業場所を固定する。ここにだけファイルを置き、変更の履歴を git に残す。ビルドで自動的にできる `a.out` などは `.gitignore` に書いて、履歴に入れない**

```
. ~/se-handson-kit/env.sh
git init -b main
cp $KIT_DIR/step0/.clang-format $KIT_DIR/step0/.gitignore .
git add -A
git commit -m "chore: .clang-format と .gitignore を追加"
git log --oneline
```

```
$ git log --oneline
5ec8524 chore: .clang-format と .gitignore を追加
(終了コード: 0)
```

`.clang-format` は、チームの規約（LLVM スタイル、インデント4、1行80桁）である。

---

### ステップ1（2分）粗削り版を書く

★⇒ **最初のコードは、動くかどうかも分からない「粗削り」でよい。まずコンパイラの警告を見て、コミットする**

```
git switch -c step1-draft
cp $KIT_DIR/step1/bubble_sort.c .
cat bubble_sort.c          # bat bubble_sort.c でもよい
$CC -Wall -Wextra -c bubble_sort.c
git add -A
git commit -m "feat: バブルソート粗削り版"
```

```
$ $CC -Wall -Wextra -c bubble_sort.c
(終了コード: 0)
```

**警告は出ない。つまり、コンパイラを通ったことは正しさを保証しない**（後のステップで効いてくる）**

--- 

### ステップ2（2.5分）フォーマッタ：clang-format

★⇒ **見た目の整形は機械に任せる。**整形しただけで挙動は変わらない**ことを、差分で確かめる**

```
git switch -c step2-format
clang-format -i bubble_sort.c
git diff
```

```
$ git diff
diff --git a/bubble_sort.c b/bubble_sort.c
index b03668b..c97cd99 100644
--- a/bubble_sort.c
+++ b/bubble_sort.c
@@ -1,12 +1,12 @@
-void bubble_sort(int n, int *a)
-{ int i, j;
-  for (i = 0; i < n; i++) {
-     for (j = 1; j < n - i; j++) {
-         if (a[j - 1] < a[j]) {
-             int t = a[j - 1];
-             a[j - 1] = a[j];
-             a[j] = t;
-         }
-    }    
-  }
+void bubble_sort(int n, int *a) {
+    int i, j;
+    for (i = 0; i < n; i++) {
+        for (j = 1; j < n - i; j++) {
+            if (a[j - 1] < a[j]) {
+                int t = a[j - 1];
+                a[j - 1] = a[j];
+                a[j] = t;
+            }
+        }
+    }
 }
(終了コード: 0)
```

ほぼ全行が変わったように見えるが、空白を無視する `-w` を付けると、実質の変更は波括弧の位置だけだと分かる。

```
git diff -w
```

```
$ git diff -w
diff --git a/bubble_sort.c b/bubble_sort.c
index b03668b..c97cd99 100644
--- a/bubble_sort.c
+++ b/bubble_sort.c
@@ -1,5 +1,5 @@
-void bubble_sort(int n, int *a)
-{ int i, j;
+void bubble_sort(int n, int *a) {
+    int i, j;
     for (i = 0; i < n; i++) {
         for (j = 1; j < n - i; j++) {
             if (a[j - 1] < a[j]) {
(終了コード: 0)
```

```
git add -A
git commit -m "style: clang-format を適用（整形のみ）"
```

---

### ステップ3（3.5分）リンタ：clang-tidy

★⇒ **`-checks='*'` は全部のチェックを有効にする指定なので、目的に合わない指摘も出る。**1件ずつ「直す・直さない」を決めて、理由を残す**ことが大事である**

```
git switch -c step3-lint
clang-tidy -checks='*' bubble_sort.c --
```

`--` より前がファイル名で、後ろはコンパイラへの引数である。

```
$ clang-tidy -checks='*' bubble_sort.c --
8 warnings generated.
~/se-handson/bubble_sort.c:1:30: warning: parameter name 'a' is too short, expected at least 3 characters [readability-identifier-length]
    1 | void bubble_sort(int n, int *a) {
      |                              ^
~/se-handson/bubble_sort.c:2:5: warning: multiple declarations in a single statement reduces readability [readability-isolate-declaration]
    2 |     int i, j;
      |     ^~~~~~~~~
~/se-handson/bubble_sort.c:2:9: warning: variable 'i' is not initialized [cppcoreguidelines-init-variables]
    2 |     int i, j;
      |         ^
note: this fix will not be applied because it overlaps with another fix
~/se-handson/bubble_sort.c:2:9: warning: variable name 'i' is too short, expected at least 3 characters [readability-identifier-length]
~/se-handson/bubble_sort.c:2:12: warning: variable 'j' is not initialized [cppcoreguidelines-init-variables]
    2 |     int i, j;
      |            ^
note: this fix will not be applied because it overlaps with another fix
~/se-handson/bubble_sort.c:2:12: warning: variable name 'j' is too short, expected at least 3 characters [readability-identifier-length]
~/se-handson/bubble_sort.c:4:9: warning: kernel performance could be improved by unrolling this loop with a '#pragma unroll' directive [altera-unroll-loops]
    4 |         for (j = 1; j < n - i; j++) {
      |         ^
~/se-handson/bubble_sort.c:6:21: warning: variable name 't' is too short, expected at least 3 characters [readability-identifier-length]
    6 |                 int t = a[j - 1];
      |                     ^
(終了コード: 0)
```

今回の判断は次のとおり（ここを学生と一緒に読む）：

|チェック名|指摘の内容|対応|理由|
|-|-|-|-|
|readability-identifier-length|引数 `a`、変数 `t` の名前が短い|直す|`a` → `arr`、`t` → `tmp`。意味が伝わる名前にする|
|readability-isolate-declaration|`int i, j;` と1文でまとめて宣言している|直す|1行に1変数にする|
|cppcoreguidelines-init-variables|`i`、`j` が未初期化|直す|`= 0` で初期化する|
|readability-identifier-length|ループ変数 `i`、`j` の名前が短い|見送り|ループ変数に `i`、`j` を使うのは慣習で、読みにくくならない|
|altera-unroll-loops|`#pragma unroll` を勧める|見送り|FPGA（Altera）向けのチェックで、この課題には無関係|

反映したファイルを置いて、もう一度リンタをかけ、差分を見る。

```
cp $KIT_DIR/step3/bubble_sort.c .
clang-tidy -checks='*' bubble_sort.c --
```

```
$ clang-tidy -checks='*' bubble_sort.c --
3 warnings generated.
~/se-handson/bubble_sort.c:2:9: warning: variable name 'i' is too short, expected at least 3 characters [readability-identifier-length]
    2 |     int i = 0;
      |         ^
~/se-handson/bubble_sort.c:3:9: warning: variable name 'j' is too short, expected at least 3 characters [readability-identifier-length]
    3 |     int j = 0;
      |         ^
~/se-handson/bubble_sort.c:5:9: warning: kernel performance could be improved by unrolling this loop with a '#pragma unroll' directive [altera-unroll-loops]
    5 |         for (j = 1; j < n - i; j++) {
      |         ^
(終了コード: 0)
```

残った3件は「見送り」と決めたものだけである。差分は次のとおり：

```
git diff
```

```
$ git diff
diff --git a/bubble_sort.c b/bubble_sort.c
index c97cd99..841a9a3 100644
--- a/bubble_sort.c
+++ b/bubble_sort.c
@@ -1,11 +1,12 @@
-void bubble_sort(int n, int *a) {
-    int i, j;
+void bubble_sort(int n, int *arr) {
+    int i = 0;
+    int j = 0;
     for (i = 0; i < n; i++) {
         for (j = 1; j < n - i; j++) {
-            if (a[j - 1] < a[j]) {
-                int t = a[j - 1];
-                a[j - 1] = a[j];
-                a[j] = t;
+            if (arr[j - 1] < arr[j]) {
+                int tmp = arr[j - 1];
+                arr[j - 1] = arr[j];
+                arr[j] = tmp;
             }
         }
     }
(終了コード: 0)
```

```
git add -A
git commit -m "refactor: clang-tidy の指摘に対応（命名・変数初期化）"
```

---

### ステップ4（2.5分）テストを書く：ここで落ちる

★⇒ **テストは「期待値を先に決めて、実際の出力と比べる」プログラムである。まず最小の1ケース（昇順にそろった配列は、そのまま変わらない）を試す**

```
git switch -c step4-test
cp $KIT_DIR/step4/bubble_test.c .
$CC -g -W -Wall bubble_test.c bubble_sort.c && ./a.out
```

```
$ $CC -g -W -Wall bubble_test.c bubble_sort.c && ./a.out
a.out: bubble_test.c:12: main: Assertion `memcmp(input1, expect1, sizeof(input1)) == 0' failed.
(終了コード: 134)
```

**失敗する。** 整形もリントも通ったのに、中身が間違っていた、という場面である。「昇順」のはずが `if (arr[j - 1] < arr[j])` で、前が小さいと入れ替える＝**降順**になっている。ここで、テストが落ちたこと自体を記録するために、失敗したままコミットしておく。

```
git add -A
git commit -m "test: 最初のテストを追加（失敗する）"
```

---

### ステップ5（2分）修正する

★⇒ **原因は比較の向き。ついでに、`NULL` や要素数0〜1の配列でも落ちないよう、先頭で戻す。差分は小さいので、`git diff` でそのまま読める**

```
git switch -c step5-fix
cp $KIT_DIR/step5/bubble_sort.c .
git diff
```

```
$ git diff
diff --git a/bubble_sort.c b/bubble_sort.c
index 841a9a3..e1add0c 100644
--- a/bubble_sort.c
+++ b/bubble_sort.c
@@ -1,9 +1,14 @@
+#include <stddef.h>
+
 void bubble_sort(int n, int *arr) {
     int i = 0;
     int j = 0;
+    if (arr == NULL || n < 2) {
+        return;
+    }
     for (i = 0; i < n; i++) {
         for (j = 1; j < n - i; j++) {
-            if (arr[j - 1] < arr[j]) {
+            if (arr[j - 1] > arr[j]) {
                 int tmp = arr[j - 1];
                 arr[j - 1] = arr[j];
                 arr[j] = tmp;
(終了コード: 0)
```

```
$CC -g -W -Wall bubble_test.c bubble_sort.c && ./a.out
```

```
$ $CC -g -W -Wall bubble_test.c bubble_sort.c && ./a.out
test succeeded
(終了コード: 0)
```

```
git add -A
git commit -m "fix: 比較の向きを修正（降順→昇順）、NULL・短い配列を早期return"
```

---

### ステップ6（4分）カバレッジ：テストが通っても安心できない

★⇒ **「テストが通った」は「テストが実行した範囲で通った」にすぎない。**どの行が一度も実行されていないか**を `gcov` で調べる**

```
git switch -c step6-coverage
$CC -c -g -W -Wall -coverage bubble_test.c bubble_sort.c
$CC -coverage bubble_test.o bubble_sort.o
./a.out
$GCOV -b bubble_sort.c       # -b は分岐カバレッジも表示
```

```
$ $CC -c -g -W -Wall -coverage bubble_test.c bubble_sort.c && $CC -coverage bubble_test.o bubble_sort.o && ./a.out && $GCOV -b bubble_sort.c
test succeeded
File 'bubble_sort.c'
Lines executed:63.64% of 11
Branches executed:100.00% of 10
Taken at least once:70.00% of 10
No calls
Creating 'bubble_sort.c.gcov'

Lines executed:63.64% of 11
(終了コード: 0)
```

テストは通ったのに、行カバレッジは **63.64%** である。どの行かは `bubble_sort.c.gcov` で分かる（`#####` が一度も実行されていない行）：

```
grep -v -e '^branch' -e '^call' bubble_sort.c.gcov
```

```
$ grep -v -e '^branch' -e '^call' bubble_sort.c.gcov
        -:    0:Source:bubble_sort.c
        -:    0:Graph:bubble_sort.gcno
        -:    0:Data:bubble_sort.gcda
        -:    0:Runs:1
        -:    1:#include <stddef.h>
        -:    2:
function bubble_sort called 1 returned 100% blocks executed 83%
        1:    3:void bubble_sort(int n, int *arr) {
        1:    4:    int i = 0;
        1:    5:    int j = 0;
        1:    6:    if (arr == NULL || n < 2) {
    #####:    7:        return;
        -:    8:    }
        6:    9:    for (i = 0; i < n; i++) {
       15:   10:        for (j = 1; j < n - i; j++) {
       10:   11:            if (arr[j - 1] > arr[j]) {
    #####:   12:                int tmp = arr[j - 1];
    #####:   13:                arr[j - 1] = arr[j];
    #####:   14:                arr[j] = tmp;
        -:   15:            }
        -:   16:        }
        -:   17:    }
        -:   18:}
(終了コード: 0)
```

入れ替えの3行と、`return` が実行されていない。昇順の配列しか試していないので、入れ替えが一度も起きなかった。逆順、負数と重複、要素1個、0個、`NULL` のテストを足す。

```
rm -f *.o *.gcda *.gcno *.gcov
cp $KIT_DIR/step6/bubble_test.c .
git diff --stat
$CC -c -g -W -Wall -coverage bubble_test.c bubble_sort.c
$CC -coverage bubble_test.o bubble_sort.o
./a.out
$GCOV -b bubble_sort.c
```

```
$ $CC -c -g -W -Wall -coverage bubble_test.c bubble_sort.c && $CC -coverage bubble_test.o bubble_sort.o && ./a.out && $GCOV -b bubble_sort.c
sorted      : PASS
reversed    : PASS
mixed       : PASS
one element : PASS
empty       : PASS
NULL        : PASS
all tests passed
File 'bubble_sort.c'
Lines executed:100.00% of 11
Branches executed:100.00% of 10
Taken at least once:100.00% of 10
No calls
Creating 'bubble_sort.c.gcov'

Lines executed:100.00% of 11
(終了コード: 0)
```

```
$ grep -v -e '^branch' -e '^call' bubble_sort.c.gcov
        -:    0:Source:bubble_sort.c
        -:    0:Graph:bubble_sort.gcno
        -:    0:Data:bubble_sort.gcda
        -:    0:Runs:1
        -:    1:#include <stddef.h>
        -:    2:
function bubble_sort called 6 returned 100% blocks executed 100%
        6:    3:void bubble_sort(int n, int *arr) {
        6:    4:    int i = 0;
        6:    5:    int j = 0;
        6:    6:    if (arr == NULL || n < 2) {
        3:    7:        return;
        -:    8:    }
       19:    9:    for (i = 0; i < n; i++) {
       51:   10:        for (j = 1; j < n - i; j++) {
       35:   11:            if (arr[j - 1] > arr[j]) {
       19:   12:                int tmp = arr[j - 1];
       19:   13:                arr[j - 1] = arr[j];
       19:   14:                arr[j] = tmp;
        -:   15:            }
        -:   16:        }
        -:   17:    }
        -:   18:}
(終了コード: 0)
```

行・分岐とも100%になった。`lcov` でも同じ値が確認できる：

```
lcov --capture --directory . --output-file cov.info --rc branch_coverage=1 --gcov-tool "$LCOV_GCOV" --quiet
lcov --extract cov.info '*/bubble_sort.c' --output-file cov_sort.info --rc branch_coverage=1 --quiet
lcov --summary cov_sort.info --rc branch_coverage=1 --gcov-tool "$LCOV_GCOV"
```

```
$ lcov --capture --directory . --output-file cov.info --rc branch_coverage=1 --gcov-tool '$LCOV_GCOV' --quiet && lcov --extract cov.info '*/bubble_sort.c' --output-file cov_sort.info --rc branch_coverage=1 --quiet && lcov --summary cov_sort.info --rc branch_coverage=1 --gcov-tool '$LCOV_GCOV'
Summary coverage rate:
  lines......: 100.0% (11 of 11 lines)
  functions..: 100.0% (1 of 1 function)
  branches...: 100.0% (10 of 10 branches)
(終了コード: 0)
```

```
git add -A
git commit -m "test: 分岐を網羅するテストを追加（行・分岐カバレッジ100%）"
```

注意点：**カバレッジ100%は、正しさの保証ではない**。実行されたかどうかしか見ていない（結果が正しいかは、テストの期待値次第）。これが、0週目以降の「ランダムテスト」へつながる。

---

### まとめ（2分）

```
git switch main
git merge --ff-only step6-coverage
git log --oneline --graph --all --decorate
git branch -v
```

```
$ git log --oneline --graph --all --decorate
* 4f4f89b (step7-random-bench) test: ランダムテストとベンチマークを追加
* a22e862 (HEAD -> main, step6-coverage) test: 分岐を網羅するテストを追加（行・分岐カバレッジ100%）
* 48c100a (step5-fix) fix: 比較の向きを修正（降順→昇順）、NULL・短い配列を早期return
* 192af6b (step4-test) test: 最初のテストを追加（失敗する）
* 6f33fc4 (step3-lint) refactor: clang-tidy の指摘に対応（命名・変数初期化）
* 17253ee (step2-format) style: clang-format を適用（整形のみ）
* c6f2abe (step1-draft) feat: バブルソート粗削り版
* 5ec8524 chore: .clang-format と .gitignore を追加
(終了コード: 0)
```

**補足説明**：

ステップごとに残したブランチを見ると、「何をしたら何が変わったか」が全部たどれる。実験のレポートも、この履歴と `git diff` を根拠にして書く。差分を見るのはいつでもできる：

```
git diff step1-draft step3-lint     # 粗削り版 → リント反映版
git diff step4-test step5-fix       # 修正の前後（バグ修正）
git diff -w step1-draft step2-format
```
---

### ステップ7（予備／各自）ランダムテストとベンチマーク

時間が余ったとき、または各自で試す。固定のテストでは試せない入力を、乱数で大量に作って `qsort` と比べる。さらに、データサイズを変えて実行時間を測る。

```
git switch -c step7-random-bench
cp $KIT_DIR/step7/*.c .
$CC -g -W -Wall bubble_randomtest.c bubble_sort.c -o rtest && ./rtest
$CC -W -Wall -O2 bubble_bench.c bubble_sort.c -o bench && ./bench
```

```
$ $CC -g -W -Wall bubble_randomtest.c bubble_sort.c -o rtest && ./rtest
trials = 10000, failures = 0
(終了コード: 0)
```

```
$ $CC -W -Wall -O2 bubble_bench.c bubble_sort.c -o bench && ./bench
size,best_time
100,0.000020111
200,0.000074484
500,0.000440965
1000,0.001732406
2000,0.006984022
5000,0.044195818
10000,0.205140779
20000,0.786877587
(終了コード: 0)
```

データサイズが2倍になると、時間は約4倍になる（10000 → 20000 で約5倍）。$O(n^2)$ らしい振る舞いである。ベンチマークは $N = 100000$ まで測ると1分以上かかるので、ここでは $N \le 20000$ に絞ってある。測定値は環境で変わる。


-----

## 4. 振り返り：今日の道具が見つけたこと

|ステップ|道具|見つけたこと|見つけられなかったこと|
|-|-|-|-|
|1|gcc -Wall -Wextra|（警告なし）|比較の向きの間違い|
|2|clang-format|見た目の不揃い|挙動|
|3|clang-tidy|名前の短さ、未初期化の変数|比較の向きの間違い|
|4|テスト|昇順にならないこと（バグ）|試していない入力の不具合|
|6|gcov／lcov|テストが実行していない行|実行された行が正しいか|

## 5. 困ったとき

- 途中で手が止まったら、そのステップの**ブランチ名を指定して正解に戻る**：
  ```
  git switch step3-lint        # 未コミットの変更があると拒否される。その場合は下の2行
  git restore .                # 作業中の変更を捨てる（戻せない）
  git switch step3-lint
  ```
- 完成版の見本は `reference-repo/` に全ブランチ入りで置いてある。
- どうしても進まない場合や、デモを一気に見せたい場合は、自動再生を使う：
  ```
  bash ~/se-handson-kit/demo-auto.sh ~/se-handson-demo
  ```
  新しいディレクトリを作り、ステップ0〜7をすべて自動で実行して、同じブランチを作る。
- `error: The following untracked working tree files would be overwritten` などと出たら、`git status` で状態を確認して、不要なファイルは消す。


- **複雑さの表（元資料の 1.3(5)）は、ハンズオンには入れていない。** 20分に収まらないので、本編の実験で扱う想定である。
- Mac 向けの手順（Homebrew、`env.sh`）は、Linux 上のリハーサルで動作を確かめたもので、**Mac 実機では未確認**である。事前に `check-env.sh` で確認してほしい。
