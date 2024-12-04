---
title: PostgreSQLのSEARCH_PATHの落とし穴
tags:
  - PostgreSQL
  - AdventCalendar2024
private: true
updated_at: '2024-12-04T18:57:01+09:00'
id: 835848f159e1e2f83d5f
organization_url_name: null
slide: false
ignorePublish: false
---
## SCHEMA

PostgreSQLには、`SCHEMA（スキーマ）`という概念があります。`SCHEMA`は、データベース内に作られるテーブルやビュー、関数、型などのデータベースオブジェクトを含む名前空間です。
テーブルは`SCHEMA`に属していて、テーブルにアクセスするにはS`CHEMA名.テーブル名`という形式で指定します。

例）

```sql
SELECT * FROM schema_name.table_name;
```

つまり、階層構造として、データベース名 > スキーマ名 > テーブル名という形になります。標準SQLにならい、その形式でも指定できます。

例）

```sql
SELECT * FROM database_name.schema_name.table_name;
```

ただし、PostgreSQLでは、接続したデータベース以外にはアクセスできません。そのため、実行しているデータベースの間違い防止ぐらいの意味しかありません。通常はデータベース名は省略します。

そして、SCHEMA名も省略できます。省略した場合には、SEARCH_PATHによって解決される...というのは、ちょっと語弊があるなというのが、この記事の主旨です。

## SEARCH_PATH

SCHEMAは複数作成でき、違うSCHEMAに同じ名前のテーブル等を作成できます。同じ名前のテーブルを作成しておいて、アクセスするときに違うSCHEMAを使用することで、実際には違うテーブルを参照できるというのは意図した動作です。

例えば、PostgreSQLのデフォルトの`SEARCH_PATH`では'$user', publicとなっています。'$user'はユーザー名のSCHEMAとなります。
`SEARCH_PATH`は、左側が優先です。
ユーザー用のテーブルをユーザー名のSCHEMAに作成し、共通のテーブルを`public`のSCHEMAに作成することで、ログインしたユーザー向けの結果を返すことができます。

```sql
SELECT * FROM user_name.table_usre LEFT JOIN public.table_public ON user_name.table_user.id = public.table_public.id;
```

を

```sql
SELECT * FROM table_user LEFT JOIN table_public ON table_user.id = table_public.id;
```

と書けることになります。記述が少なくなるだけでなく同じSQLでユーザー毎の結果を返すことができます。

そして、SEARCH_PATHは、`SET SEARCH_PATH`で設定できます。デフォルトの`SEARCH_PATH`は、`SHOW SEARCH_PATH`で確認できます。

```sql
# CREATE SCHEMA test;
CREATE SCHEMA
# SET SEARCH_PATH = test;
SET
# show SEARCH_PATH;
 search_path 
-------------
 test
(1 row)
```

最近のPostgreSQLでは、デフォルトの`public SCHEMA`の権限が見直されたり、使用についての説明が詳細にされていたりするのでマニュアルを参照してください。[PostgreSQL: Documentation: 16: 5.7. スキーマ#5.9.6. 使用パターン](https://www.postgresql.jp/document/16/html/ddl-schemas.html#DDL-SCHEMAS-PATTERNS)

## 探すのはSEARCH_PATHだけではない

SEARCH_PATHが設定されたSCHEMAだけが使われるというシンプルな世界であれば良いのですが、実際にはそう単純ではありません。
その要因は`pg_catalog SCHEMA`と`pg_temp SCHEMA`です。

`pg_catalog SCHEMA`は、マニュアルから引用すると

> このスキーマにはシステムテーブルと全ての組み込みデータ型、関数および演算子が含まれています。pg_catalogは常に検索パスに含まれています。

となっています。
これが、もし`検索パス`にpg_catalogが含まれて**いない**場合は、組み込み関数が`SELECT pg_catalog.count(*) FROM test;`のように指定しないと使えなくなってしまいますので、必要な措置です。

さらにややこしいのが`pg_temp SCHEMA`です。PostgreSQLでは、一時テーブルを作成するときには、通常のテーブルが作成される`SCHEMA`とは違って、`pg_temp SCHEMA`に作成されます。

```sql
CREATE TEMP TABLE test (id int);
```

とすると、`pg_temp SCHEMA`に`test`テーブルが作成されます。このままだと`pg_temp.test`としてアクセスする必要があるため、自動で`pg_temp SCHEMA`が検索パスに追加されます。それにより、

```sql
SELECT * FROM test;
```

と書けるようになります。そして`pg_temp SCHEMA`は、通常の検索パスより優先されるため、通常の`test`テーブルがあっても`pg_temp.test`が参照されます。これは通常テーブルと同じ名前の一時テーブルを作成しても、一時テーブルが参照されるようにするという使い方をするためだと理解できます。

## SEARCH_PATHの優先順位

ということで、`SCHEMA`を省略した場合は`SEARCH_PATH`に加えて`pg_temp`と`pg_catalog`が探されるということになります。
上記に書いたように`SCHEMA`が違えば同じ名前のテーブルを作成できるため、`SCHEMA`を省略した場合は、優先順位が重要になります。

デフォルトでは、`pg_temp`、`pg_catalog`、`SEARCH_PATH`に書いた順（SERCH_PATHのデフォルトは'$user', public）となります。

しかーし、`SEARCH_PATH`に`pg_temp`や`pg_catalog`を加えると優先順位を変えることができ、例えば`public`を優先することができます。
つまり関数のオーバーライドができるということです。

まず通常のSQL関数を作ります。max()は集約関数ですが、今回は単純な文字列を返す関数にしてみます。これは`SCHEMA`を指定していないため、デフォルトの`public`に作成されます。

```sql
# CREATE function max(integer) RETURNS text AS $$
SELECT 'TORA TORA TORA';
$$ LANGUAGE SQL IMMUTABLE;
```

通常のSEARCH_PATHで実行。

```sql
# SHOW SEARCH_PATH; 
   search_path   
-----------------
 "$user", public
(1 row)

SELECT max(1);
max 
-----
   1
(1 row)
```

`public`を優先するように`SEARCH_PATH`を変更して実行。

```sql
# SET search_path = public, pg_catalog;
SET
SELECT max(1);
      max       
----------------
 TORA TORA TORA
(1 row)
```

作成したmax()関数が実行されました。

さて、上記で書いたデフォルトの優先順位を思い出して下さい。
`pg_temp`、`pg_catalog`、`SEARCH_PATH`の順です。つまり、`pg_temp`に同じ名前の関数を作成すると、`pg_temp`の関数が実行されることになりませんか？

```sql
# CREATE function pg_temp.max(integer) RETURNS text AS $$
SELECT 'Give me a Shake';
$$ LANGUAGE SQL IMMUTABLE;
```

SEARCH_PATHをデフォルトに戻して実行すると`pg_temp`、`pg_catalog`、`SEARCH_PATH`の順のはず...

```sql
# SHOW search_path ;
   search_path   
-----------------
 "$user", public
(1 row)
```

```sql
SELECT max(1);
max 
-----
   1
(1 row)
```

ですが、**安心して下さい。なりません。**

[PostgreSQLのマニュアル](https://www.postgresql.jp/document/16/html/runtime-config-client.html#GUC-SEARCH-PATH) から引用すると

> 同様に、現在のセッションの一時テーブルスキーマpg_temp_nnnも、存在すれば常に検索されます。 これはpg_tempという別名を使用してパスに明示的に列挙させることができます。 パスに列挙されていない場合、最初に（pg_catalogよりも前であっても）検索されます。 しかし、一時スキーマはリレーション（テーブル、ビュー、シーケンスなど）とデータ型名に対してのみ検索されます。 **関数や演算子名に対してはまったく検索されません。**

となっていて、`pg_temp`に関数を作成しても`SCHEMA`を指定しない限り実行されないということです。

関数や演算子名に関しては、まあ良いことにしても`SEARCH_PATH`の優先順位はややこしいです。
さらに、`SEARCH_PATH`のセットにミスっても誰も指摘してくれません。
環境変数的なものなので、そういうものなのですが。

```sql
# SET search_path = 'public, pg_catalog, pg_temp'
SET
# SHOW search_path;
          search_path          
-------------------------------
 "public, pg_catalog, pg_temp"
(1 row)
```

と一見問題になさそうですが、実際には`SCHEMA`の設定がされていないので、`pg_temp`, `pg_catalog`の順で検索されてしまいます。

そのため、`SEARCH_PATH`を変更するときは、身を清め、白装束に着替え、一点に集中してミスの無いようにセットする必要があります（ウソです）。

`SEARCH_PATH`がほんとのところはどうなっているか確認するには、`current_schemas()`関数を使います。

```sql
SELECT current_schemas(true);
        current_schemas        
-------------------------------
 {pg_temp_1,pg_catalog}
(1 row)
```

となって、search_pathに'public, pg_catalog, pg_temp'という文字列で渡してしまったことに気づけます（実際には`SET search_patth = public, pg_catalog, pg_temp`と書く必要があった）。

`pg_temp_1`は、実際のスキーマ名です。pg_tempはセッションごとに異なりますが、サーバー側で名前が一意になるようにされるため、`pg_temp_1`のような名前がつけられ、`pg_temp`としてもアクセスできるようになります。

ということで、ほんとのところは`current_schemas(true)`関数を使って確認してから、リレーション（テーブル、ビュー、シーケンス）は探す、関数や演算子名は`pg_temp_*`を除外して探す...ということになります。

入口はシンプルそうに見えて、中はややこしい世界ですね。
