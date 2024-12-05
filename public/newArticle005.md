---
title: psqlのページャーにovを！
tags:
  - 'PostgreSQL'
  - 'AdventCalendar2024'
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
この記事は、[2024年Advent Calendar](https://qiita.com/advent-calendar/2024/postgresql)の12月6日の記事その2です。

## psqlのページャー

psqlではSQLを実行して結果が画面に収まらない場合、ページャーが起動して結果を表示します。
デフォルトのページャーは`less`ですが、`ov`を使うことで、`less`の代わりに`ov`を使うことができます。
