---
title: xlsxファイルにSQLを実行するxlsxsql
tags:
  - Excel
  - XLSX
  - trdsql
private: true
updated_at: '2023-11-23T12:11:46+09:00'
id: f93196b4600aacefd8f0
organization_url_name: null
slide: false
ignorePublish: false
---
# xlsxファイルにSQLを実行するxlsxsql

xlsxファイルに対してSQLを実できるツールを作りました。

[GitHub](https://github.com/noborus/xlsxsql)

## これは何？

xlsxsqlは、xlsxファイルに対してSQLを実行するツールです。
また、CSV,LTSV,JSON,YAMLといったファイルに対してSQLを実行することもでき、その結果をxlsxファイルに出力することもできます。

[trdsql](https:://github.com/noborus/trdsql)にxlsxファイルの読み書き機能を追加したものになります。

## 使い方

単純にファイルをテーブルとして指定できます。

```sh
$ xlsxsql -o CSV "SELECT * FROM test.xlsx"
id,name
1,apple
2,orange
3,melon
```

最初の行をヘッダーとして扱う場合は -H オプションを付けます。

```sh
$ xlsxsql -o JSON -H "SELECT * FROM test.xlsx"
```

```json
[
  {
    "id": "1",
    "name": "apple"
  },
  {
    "id": "2",
    "name": "orange"
  },
  {
    "id": "3",
    "name": "melon"
  }
]
```

### シート名の指定

ファイルを指定しただけの場合は、最初のシート全体が対象となります。

シート名を指定する場合は、ファイル名のあとに`::`を付けてシート名を付けます

```sh
$ xlsxsql -o CSV "SELECT * FROM test.xlsx::Sheet2"
```

シートのリストは `list`サブコマンドで確認できます。

```sh
$ xlsxsql list test.xlsx
Sheet1
Sheet2
```

### セルの指定

セルを指定することもできます。セルを指定した場合は、値が一つもない行または値が一つもない列に達するまでをテーブルとして扱います。
セルはシート名のあとに`.`を付けて指定します。

```sh
$ xlsxsql -o CSV "SELECT * FROM test.xlsx::Sheet2.B2"
```

シート名を省略した場合は、最初のシートが対象となります。'::.セル名'と書けます。

```sh
$ xlsxsql -o CSV "SELECT * FROM test.xlsx::.B2"
```

### xlsxファイルに出力

`-o`オプションで出力ファイル形式を指定できますが、xlsxファイルに出力する場合はファイル名が必要になります。
さらにファイル名は`.xlsx`で終わる必要があります。そのため出力ファイル名を指定するオプション`--out-file`に`.xlsx`のついたファイル名を指定すれば
`-o xlsx`は省略できます。

```sh
$ xlsxsql --out-file out.xlsx "SELECT * FROM test.xlsx"
```

また、[trdsql](https:://github.com/noborus/trdsql)で使用できるファイル形式は全て使用できます。そのため、`CSV`ファイルを`xlsx`ファイルに変換することもできます。

```sh
$ xlsxsql --out-file out.xlsx "SELECT * FROM test.csv"
```

#### シートのクリア

指定したファイルが無かった場合は、新規に作成されます。ファイルがあった場合は、そのファイルに出力されます。単純に元のファイルのセルに埋めていくだけなので、出力分だけ上書きされ、それ以外のセルは変更されません。

書き出すシートを出力する内容だけにしたい場合は、`--clear-sheet`オプションを付けます。

#### シートの指定、セルの指定

`xlsx`ファイルに出力する場合は、シート名を指定できます（デフォルトはSheet1です）。また、セルを指定することもできます。

```sh
$ xlsxsql --out-file out.xlsx --out-sheet Sheet3 --out-cell F6 --out-header "SELECT * FROM test.csv"
```

Note: `--out-header`オプションを付けると、ヘッダー行も出力します。

### SQL文の省略

`SELECT * FROM`はよく使われるので、`table`サブコマンドが用意されています。（`table`は`SELECT * FROM`の省略形です）

```sh
$ xlsxsql -o AT -H table test.xlsx
+----+--------+
| c1 |   c2   |
+----+--------+
|  1 | Orange |
|  2 | Melon  |
|  3 | Apple  |
+----+--------+
```

Tips: `AT`は`ASCII Table`の出力です。

### JOIN

JOINもできます。これまでの指定を組み合わせてみます。

まずシートの全体をテーブルで確認します。

```sh
$ xlsxsql -o AT table test3.xlsx
+----+----+--------+--------+----+----+-------+
| A1 | B1 |   C1   |   D1   | E1 | F1 |  G1   |
+----+----+--------+--------+----+----+-------+
|    |    | id     | name   |    |    |       |
|    |    |      1 | apple  |    |    |       |
|    |    |      2 | orange |    |    |       |
|    |    |      3 | melon  |    | id | price |
|    |    |        |        |    |  1 |   100 |
|    |    |        |        |    |  2 |    50 |
|    |    |        |        |    |  3 |   500 |
+----+----+--------+--------+----+----+-------+
```

C1からD4までにid,nameのテーブルがあり、F4からG7までにid,priceのテーブルがあります。

この２つのテーブルをidで1でJOINしてみます。

```sh
$ xlsxsql -o AT -H query "SELECT a.id,a.name,b.price 
FROM test3.xlsx::.C1 AS a 
LEFT JOIN test3.xlsx::.F4 AS b ON a.id=b.id"
+----+--------+-------+
| id |  name  | price |
+----+--------+-------+
|  1 | apple  |   100 |
|  2 | orange |    50 |
|  3 | melon  |   500 |
+----+--------+-------+
```

もちろん、この内容をxlsxファイルに出力できます。入力のファイル名を出力のファイル名にすれば、同じファイルに出力できます。

```sh
$ xlsxsql --out-file test3.xlsx --out-header --out-sheet Sheet1 --out-cell B10 -H query 
"SELECT a.id,a.name,b.price 
FROM test3.xlsx::.C1 AS a 
LEFT JOIN test3.xlsx::.F4 AS b ON a.id=b.id"
```

もう一度、シート全体を確認すると以下のようになります。

```sh
$ xlsxsql -o AT table test3.xlsx
+----+----+--------+--------+----+----+-------+
| A1 | B1 |   C1   |   D1   | E1 | F1 |  G1   |
+----+----+--------+--------+----+----+-------+
|    |    | id     | name   |    |    |       |
|    |    |      1 | apple  |    |    |       |
|    |    |      2 | orange |    |    |       |
|    |    |      3 | melon  |    | id | price |
|    |    |        |        |    |  1 |   100 |
|    |    |        |        |    |  2 |    50 |
|    |    |        |        |    |  3 |   500 |
|    |    |        |        |    |    |       |
|    |    |        |        |    |    |       |
|    | id | name   | price  |    |    |       |
|    |  1 | apple  |    100 |    |    |       |
|    |  2 | orange |     50 |    |    |       |
|    |  3 | melon  |    500 |    |    |       |
+----+----+--------+--------+----+----+-------+
```

CSVファイルとのJOINも可能です。

別にCSVファイルがあるとします。

```csv
id,price
1,120
2,130
3,140
```

さきほどのxlsxファイルとCSVファイルをJOINしてみます。`test3.xlsx::.F4`をCSVファイルに変更します。

```sh
xlsxsql --out-file test3.xlsx --out-header --out-sheet Sheet1 --out-cell B10 -H query 
"SELECT a.id,a.name,b.price 
FROM test3.xlsx::.C1 AS a 
LEFT JOIN test.csv AS b ON a.id=b.id"
```

結果は以下のようになります。

```sh
$ xlsxsql -o MD table test3.xlsx
```

Tips: MDはMarkdown Table形式の出力です。

| A1 | B1 |   id   |  name  | E1 | F1 |  G1   |
|----|----|--------|--------|----|----|-------|
|    |    |      1 | apple  |    |    |       |
|    |    |      2 | orange |    |    |       |
|    |    |      3 | melon  |    | id | price |
|    |    |        |        |    |  1 |   100 |
|    |    |        |        |    |  2 |    50 |
|    |    |        |        |    |  3 |   500 |
|    |    |        |        |    |    |       |
|    |    |        |        |    |    |       |
|    | id | name   | price  |    |    |       |
|    |  1 | apple  |    120 |    |    |       |
|    |  2 | orange |    130 |    |    |       |
|    |  3 | melon  |    140 |    |    |       |

## まとめ

xlsxsqlのソースコードはそれほど大きくなく、trdsqlに機能追加しただけですが、SQLの強力な構文に加えてCSV,JSON,YAML等の変換ができるので、規模の割にできることが多いツールになっていると思います。
