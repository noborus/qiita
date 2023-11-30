---
title: 0()から始まるPostgreSQL
tags:
  - SQL
  - PostgreSQL
private: false
updated_at: '2023-12-01T00:05:39+09:00'
id: 5e39d144b1510f6865bd
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

これはPostgreSQLを0から学ぶための入門記事ではありません。
誤解を招くタイトルで申し訳ありません。

PostgreSQLで0にまつわる話を書いていきます。

## 最小のテーブル構成

SQLはテーブルを（複数の）対象として、1つのテーブルを求めるための言語です。
SQLが対象とするテーブルの構造は、行(row)と列(column)で構成されています。

| 列1 | 列2 | 列3 |
| --- | --- | --- |
| 行1 | 行1 | 行1 |
| 行2 | 行2 | 行2 |
| 行3 | 行3 | 行3 |

最小のテーブル構成は何でしょうか？理論上は0行0列のテーブルが最小のテーブル構成となります。この場合の0は、プログラミング言語でいう配列の一番目[0]ではなく、長さが0の配列[]のようなものです。

### 行0のテーブル

行が0のテーブルは、ごく普通に存在します。通常の検索で、検索条件にヒットしなければ、0行の結果テーブルが返されますし、`CREATE TABLE`で作成しただけのテーブルは、行が0のテーブルです。

### 列0のテーブル

列が0のテーブルは、通常見ることはありません。列が0のテーブルは、値を持てないので、わざわざ作る必要がないからです。
標準のSQLでは、列が0のテーブルを作ることはできないらしく、大抵のRDBでも列が0のテーブルは作れません。

しかしPostgreSQLでは、列が0のテーブルを作ることができます。

```sql
CREATE TABLE zero_column ();
```

PostgreSQL 7.4から列が0のテーブルを作成できることが明示されています。
考え方としては、`ALTER TABLE テーブル DROP COLUMN 列名`で列を削除することで列が0のテーブルを作成できてしまうため、`DROP COLUMN`で列があったとしても、列数が0にならないようにチェックするか、列が0になるのを許容するかで、PostgreSQLは後者を選択したようです。

なので、他のプログラミング言語ではよくおこなわれているような、不定長の配列（スライス）をまず空で作成して、後から要素を追加していくのが便利なように、
PostgreSQLでも理論上は、空のテーブルを作成したあとに、列を追加していくといったこともできます。
  
  ```sql
CREATE TABLE add_user ();
ALTER TABLE add_user ADD COLUMN id SERIAL;
ALTER TABLE add_user ADD COLUMN name TEXT;
ALTER TABLE add_user ADD COLUMN age INT;
```

まあ、いろんな事情により、通常やらないですし、DBAにとっては悪夢のように見えるかもしれません...

### INSERTは0列に対応しているか？

そして、ちょっと意味がわからないかもしれませんが、列が0のテーブルに行を追加することもできます。
とはいえ、`INSERT INTO zero_column VALUES ();`や`INSERT INTO zero_columns () VALUES ();`では追加できません。
どうやって書くのかわかりますか？折りたたんでおくので、考えてみてください。

<details><summary>0列に行挿入</summary>

```sql
INSERT INTO zero_column DEFAULT VALUES;
```

</details>

さらに、`SELECT`でリストを空にもできるようになっています(これはPostgreSQL 9.4から)。

```sql
SELECT FROM anther_table;
```

ということで、0列のテーブルに行を追加するもう一つの方法です。

<details><summary>0列に行挿入する別の方法</summary>

```sql
INSERT INTO zero_column SELECT FROM anther_table;
```

これにより0列複数行のテーブルが作成できます。

</details>

### 最小のSELECT

`SELECT`文は`FROM`句が省略できることが知られています。`SELECT`のリストも空にできるので、最小の`SELECT`は以下のようになります。

```sql
SELECT;
```

これをpsqlで実行すると、以下のようになります。

```sql
postgres=# SELECT;
--
(1 row)
```

結果に疑問を持たれた方もいるかもしれません。`SELECT;`の結果は0列0行のテーブルではなく、0列1行(row)のテーブルです。
`SELECT`文で`FROM`句を省略する場合、通常は`SELECT 'TEST', 1+2`のような定数を返すために使われます。
つまり、`SELECT`文で`FROM`句を省略した場合は1行（以上）のテーブルを生成すると考えられるため、0列1行のテーブルが返されます。

行を0にする方法は当然あるので、0列0行を返したいときには（いくつか方法がありますが）、私が思いついたのは以下です。

<details><summary>0列0行を返すSELECT</summary>

```sql
SELECT WHERE false;
SELECT LIMIT 0;
```

0列1行よりも0列0行を返す方が長くなるのは、面白いですね。

</details>

:::note info
ちなみに`psql`では列が0のテーブルは（改行も含めて）表示されないようになっているようです。

```sql
SELECT FROM zero_column ;
--
(6 rows)
```

これはクライアントの表示の仕方の問題で、拙作の[trdsql](https://github.com/noborus/trdsql)でJSONで出力してみると、以下のように表示されます。

```console
$ trdsql -driver postgres -ojson "SELECT FROM zero_column"
[
  {},
  {},
  {},
  {},
  {},
  {}
]
```

:::

PostgreSQLは0列に対応していますが、SQLの標準では（内部的以外は）0列に対応していないため、0列に対応しているかは構文によります。
以下はそれを見ていきます。

### SELECT COUNTは0列に対応しているか？

`SELECT`のリストは空にできますが、COUNT関数は空にできません。

```sql
SELECT count() FROM zero_column ;
ERROR:  42809: count(*) must be used to call a parameterless aggregate function
LINE 1: SELECT count() FROM zero_column ;
               ^
LOCATION:  ParseFuncOrColumn, parse_func.c:788
```

COUNT(*)ではちゃんと行数が返されます。

```sql
SELECT count(*) FROM zero_column
 count 
-------
     6
(1 row)
```

`*`は列のリストの全部を表すと思ってしまいますが、もう少し複雑です。マニュアルには以下のように書かれています。

> https://www.postgresql.jp/document/current/html/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS
> アスタリスク（*）は、いくつかの文脈において、テーブル行や複合型の全てのフィールドを表現するために使用されます。 また、集約関数の引数として使われる場合も特殊な、つまり、その集約が明示的なパラメータをまったく必要としないという意味を持ちます。

COUNTは集約関数なので、特殊な意味を持つことになります。

### VALUESは0列に対応しているか？

PostgreSQLでは`VALUES`は単体でも使用できます。`VALUES`は、`SELECT`文の`FROM`句を省略した場合と同じように、定数をテーブルにするために使われます。

```sql
VALUES(1,'a');
 column1 | column2 
---------+---------
       1 | a
(1 row)
```

ということは、`VALUES`を使えば、0列のテーブルを作成できそうと思ったのですが、エラーになりました。

```sql
VALUES();
ERROR:  42601: syntax error at or near ")"
LINE 1: values();
               ^
LOCATION:  scanner_yyerror, scan.l:1176
```

`INSERT INTO zero_column VALUES ();`がエラーになった理由がわかりましたね。

### DELETEは0列に対応しているか？

`DELETE`は`WHERE`を省略すると、テーブルの全行を削除するので問題なく動作します。

```sql
DELETE FROM zero_column ;
DELETE 6
```

では、0列が6行あるとして3行目を削除できるか？

...お前は何を言っているんだ？...案件ですが、最後の手を使って書いてみました。

<details><summary>0列の3行目を消すDELETE</summary>

```sql

SELECT ctid FROM zero_column ;
  ctid  
--------
 (0,7)
 (0,8)
 (0,9)   <-- これを消したい 
 (0,10)
 (0,11)
 (0,12)
(6 rows)
```

```sql
DELETE FROM zero_column WHERE ctid = '(0,9)';
DELETE 1
```

他に思いついたのは、ことごとく失敗しました。

</details>

### UPDATEは0列に対応しているか？

さすがに意味わからん。ムーリー。

### まとめ

```sql
SELECT FROM オチ;
--
(1 rows)
```
