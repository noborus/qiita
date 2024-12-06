---
title: MySQLShellのページャーにovを！
tags:
  - MySQL
  - AdventCalendar2024
private: true
updated_at: '2024-12-05T22:29:22+09:00'
id: e4b2d5437c4b1888f4b0
organization_url_name: null
slide: false
ignorePublish: false
---
この記事は、[2024年Advent Calendar](https://qiita.com/advent-calendar/2024/mysql)の12月8日の記事です。

## MySQLShellのページャー

## lessページャー

MySQLShellではSQLを実行して結果が画面に収まらない場合、ページャーが起動して結果を表示します。

## その前にpspg

`pspg`を使うこともできます。
[pspg](https://github.com/okbob/pspg)

## ovページャー

[ov](https://github.com/noborus/ov)は、私がGo言語で作成しているページャーです。
