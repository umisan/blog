+++
date = '2025-01-02T15:42:00+09:00'
draft = false
title = 'クエリの特徴量の計算'
mathjax = true
tags = ['conjunctive query', 'linear programming', 'python']
+++

## はじめに
conjunctive queryの特徴量として、fractional edge packing numberとfractional edge covering numberがあります。
これらの特徴量は、クエリのワーストケースの上界やアルゴリズムの上界・下界に関連しています。
今回は、これらの特徴量を計算するPythonのスクリプトを書いてみようと思います。

## 特徴量の定義
あるクエリ$q = (V, E)$のfractional edge packingとfractional edge coverはある種の制約を満たす関数$f: e \in E \\rightarrow [0,1]$です（$V$は変数の集合で、$E$はリレーションの集合です。）。
fractional edge packingが満たすべき制約は以下の通りです。
$$
\\forall v \\in V: \sum_{e: v \in e} f(e) \le 1
$$
すなわち、各変数についてその変数を含むリレーションを入力とした関数の出力の和が$1$以下である必要があります。
fractional edge coverが満たすべき制約は以下の通りです。
$$
\\forall v \\in V: \sum_{e: v \in e} f(e) \ge 1
$$
fractional edge coverでは反対に関数の出力の和が$1$以上である必要があります。

上記の制約を満たす関数は複数存在します。
制約を満たすfractional edge packingの中で、$\sum_{e \in E} f(e)$が最大なものをoptimalなfractional edge packingと呼び、その際の$\sum_{e \in E} f(e)$をfractional edge packing numberと呼びます。
これは下記の線形計画法で表すことができます。
$$
\\begin{align}
\\text{maximize: } & \\\
& \\sum_{e \\in E} f(e) \\\
\\text{subject to: } & \\\
& \\forall v \\in V: \sum_{e: v \in e} f(e) \le 1
\\end{align}
$$
論文によってはfractional covering numberとなっていますが、これは上記の問題の双対問題がfractional vertex coverとなっているからです。

fractional edge coverについては、反対に$\sum_{e \in E} f(e)$が最小なものをoptimalなfractional edge coverと呼び、その際の$\sum_{e \in E} f(e)$をfractional edge covering numberと呼びます。
こちらも同様に線形計画法で表すことができます。
$$
\\begin{align}
\\text{maximize: } & \\\
& \\sum_{e \\in E} f(e) \\\
\\text{subject to: } & \\\
& \\forall v \\in V: \sum_{e: v \in e} f(e) \ge 1
\\end{align}
$$
fractional edge covering numberはクエリの出力サイズの上界に関連しており、これをAGM boundと呼びます。
この上界については今後記事にまとめたいと思います。

## 準備
特徴量を計算するためにはLPを解く必要があるため、まずは必要なライブラリをインストールします。
Pythonには様々なライブラリが存在するようですが、今回は解くことができれば良いため特に吟味はせず[Python-MIP](https://www.python-mip.com/)を利用します。
公式ドキュメントにしたがって、インストールのために以下のコマンドを実行します。
```text
pip install mip
```
私の環境だと以下のエラーが出たため、microsoftのbuild toolをインストールします。
```text
distutils.errors.DistutilsPlatformError: Microsoft Visual C++ 14.0 or greater is required. Get it with "Microsoft C++ Build Tools": https://visualstudio.microsoft.com/visual-cpp-build-tools/
```
気を取り直してもう一度インストールを実行すると以下のエラーが出ました。
githubのprを見る限りpythonのバージョンが高すぎるとうまくいかなそうなので、3.13 -> 3.11にダウングレードし解決しました。
```text
error LNK2001: 外部シンボル _PyErr_WriteUnraisableMsg は未解決です
```
Python-MIPをインストールできたら準備は完了です。
LPを解くライブラリでは、問題をモデリングするモデラーとLPの解を計算するソルバーに分かれているようですが、サービス開発をしているわけではないので、一旦Python-MIPをインストールするとデフォルトで利用できるソルバーであるCBCを利用します。

正常にインストールできたことを確認するために、クイックスタートを行ってみます。
インタープリタを起動し、以下のコードを実行してみます。
```python
from mip import *
m = Model()
```
すると、以下の様なエラーが出ました。
```text
NameError: name 'cbclib' is not defined
```
調査すると、mipのバージョンを指定しないと問題があるようだったので、一度Python-MIPとMIPと同時にインストールされた依存ライブラリを削除し再度インストールを実行します。
```text
pip -m uninstall mip
pip uninstall cffi
pip uninstall pycparser
pip install mip==1.16rc0
```
正常にインストールが完了したのちに、再度コードを実行すると無事エラーは解消されたことが確認できました。

## 特徴量の計算
ここからは実際に特徴量を計算してみます。
まずは、以下のクエリを対象にfractional edge packing numberを計算してみます。
$$
\\begin{equation}
q = S_1(A, B, C), S_2(D, E, F), S_3(A, D), S_4(B, E), S_5(C, F)
\\end{equation}
$$
モデリングしたコードは以下の通りです。
```python
from mip import *

# packingは最大化問題
m = Model(sense=MAXIMIZE)

# 変数の追加
relations = [m.add_var(name="S{}".format(i + 1), lb = 0, ub = 1) for i in range(5)]

# 制約条件の設定
m += relations[0] + relations[2] <= 1
m += relations[0] + relations[3] <= 1
m += relations[0] + relations[4] <= 1
m += relations[1] + relations[2] <= 1
m += relations[1] + relations[3] <= 1
m += relations[1] + relations[4] <= 1

# 目的関数の設定
m.objective = xsum(relations[i] for i in range(5))

# 実行
status = m.optimize(max_seconds=300)
if status == OptimizationStatus.OPTIMAL:
    print('optimal solution {} found'.format(m.objective_value))
    print('solution:')
    for v in m.vars:
        if abs(v.x) > 1e-6: # only printing non-zeros
            print('{} : {}'.format(v.name, v.x))
else:
    print('optimal solution is not found')
```
出力結果は以下の通りです。
```text
optimal solution 3.0 found
solution:
S3 : 1.0
S4 : 1.0
S5 : 1.0
```
この結果より、$S_3, S_4, S_5$に$1$を割り当て、残りのリレーションには$0$を割り当てることが最適解であることが分かります。

続いて同じクエリのfractional edge covering numberの計算もしてみます。
制約条件以外はfractional edge packingと変わりません。
```python
from mip import *

# coverは最小化問題
m = Model(sense=MINIMIZE)

# 変数の追加
relations = [m.add_var(name="S{}".format(i + 1), lb = 0, ub = 1) for i in range(5)]

# 制約条件の設定
m += relations[0] + relations[2] >= 1
m += relations[0] + relations[3] >= 1
m += relations[0] + relations[4] >= 1
m += relations[1] + relations[2] >= 1
m += relations[1] + relations[3] >= 1
m += relations[1] + relations[4] >= 1

# 目的関数の設定
m.objective = xsum(relations[i] for i in range(5))

# 実行
status = m.optimize(max_seconds=300)
if status == OptimizationStatus.OPTIMAL:
    print('optimal solution {} found'.format(m.objective_value))
    print('solution:')
    for v in m.vars:
        if abs(v.x) > 1e-6: # only printing non-zeros
            print('{} : {}'.format(v.name, v.x))
else:
    print('optimal solution is not found')
```
出力結果は以下の通りです。
```text
optimal solution 2.0 found
solution:
S1 : 1.0
S2 : 1.0
```
この結果よりfractional edge coverではfractional edge packingとは異なり、$S_1, S_2$に1を割り当てることが最適であることがわかります。

## まとめ
本記事では、conjunctive queryの特徴であるfractional edge packing numberとfractional edge covering numberをPython-MIPを用いて計算してみました。
線計画法のモデリングより環境構築に時間が掛かってしまいましたが、モデリング自体は比較的簡単に行えました。
大きなクエリインスタンスになってくると、手動で各種numberを計算するのは最適性に疑問が出てくるため今後はPython-MIPを用いて計算してこうと思います。