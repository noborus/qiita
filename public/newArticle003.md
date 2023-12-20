---
title: tcellを使ってアプリケーションを作る
tags:
  - 'tcell'
  - 'go'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: true
---
## はじめに

この記事は、[2023年Advent Calendar](https://qiita.com/advent-calendar/2023)の[12月21日](https://qiita.com/advent-calendar/2021/12/21)の記事です。

私はGoで[ov](https://github.com/noborus/ov)というターミナルページャーをまじめに作っています。
ovは[tcell](https://github.com/gdamore/tcell)を使って構築されていて、tcellを使い倒している方だと思うので、tcellの使い方を解説します。

## GoのTUIライブラリ

その前に、GoのTUIライブラリについて説明します。

Goでターミナル上でアプリケーションを作る（いわゆるTUIアプリケーション）には、以下のライブラリが多く使われています。

* [charmbracelet/bubbletea](https://github.com/charmbracelet/bubbletea)
* [gdamore/tcell](https://github.com/gdamore/tcell) / [rivo/tview](https://github.com/rivo/tview)
* [nsf/termbox-go](https://github.com/nsf/termbox-go)

termbox-goは、現在は開発が止まっていますので、新しく作る場合はbubbleteaかtcellにtviewを組み合わせて使うのが良いと思います。

どちらもウィジットが用意されていて豊富なサンプルもありますので、自分が使いたいアプリケーションに合うものを選ぶと良いでしょう。

そういったウィジットではなく、ターミナル上で自由に描画したい場合は、tcellを直接使うことになります。ライブラリを使用せずに直接という選択肢も無いわけではありませんが、多くのターミナルエミュレーターで動作するようにするのは大変ですので、tcellを使うのが良いと思います。

## tcellとは

tcellは、ターミナルエミュレーターの差異を吸収して、描画とイベント処理を行ってくれるライブラリです。描画に関しては、端末上に直接1文字1文字置いていけるので、~~面倒くさい~~、自由度の高い描画ができます。

## tcellの使い方

tcellの使い方は、まずNewScreen()で画面を開き、Init()で初期化します。

screenに対してSetContent()で文字を置きます。

そしてイベントをPollEvent()で取得して、イベントに応じて処理をおこないShow()で画面を更新します。この繰り返しがメインの処理です。

最後はFini()で終了します。

以下が基本的な例です。

```go
package main

import (
	"github.com/gdamore/tcell/v2"
)

func main() {
	// 画面を開く
	screen, err := tcell.NewScreen()
	if err != nil {
		panic(err)
	}
	if err := screen.Init(); err != nil {
		panic(err)
	}
	defer screen.Fini()

	// イベントを処理する
	eventLoop(screen)
}

// イベントループ
func eventLoop(screen tcell.Screen) {
	for {
		// イベントを取得する
		ev := screen.PollEvent()
		switch ev := ev.(type) {
		case *tcell.EventKey:
			// キーイベントの場合
			if ev.Key() == tcell.KeyEscape {
				// ESCキーが押されたら終了する
				return
			} else {
				otherEvent(screen)
			}
		case *tcell.EventResize:
			// リサイズイベントの場合
			screen.Sync()
		}
		// 画面を更新する
		screen.Show()
	}
}

var str []rune = []rune("Hello, world!")
var x = 0

// その他のイベント
func otherEvent(screen tcell.Screen) {
	if x >= len(str) {
		x = 0
		screen.Clear()
	}
	screen.SetContent(x, 0, str[x], nil, tcell.StyleDefault)
	x++
}
```

このプログラムではESCキー以外のキーが押されると、Hello, world!を1文字ずつ表示します。ESCキーが押されると終了します。

![tcell-1.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/18555/ce4fd6da-946f-f738-6b6b-8634307da6b7.gif)

tcellでは、新しい画面が開かれると、ターミナル全体がキャンバスのようになり、終了すると元の画面にもどります（Windowsなど一部のターミナルエミュレーターでは、元の画面に戻らない場合があります）。

## 画面の描画

画面の描画は、SetContent()で行います。
引数は、x座標、y座標、メインの文字(rune)、結合文字(rune)、スタイルです。

端末より大きい座標に描画しても描画されません。そのため本来はscreen.Size()でサイズを取得して、その範囲内に描画する必要があります。

メインの文字はrune単位で指定します。ターミナルエミュレーターが対応している場合は、日本語などの表示も問題ありません。
ただ、倍の幅を持つ文字（いわゆる全角文字）をセットしたときには、右隣の座標を空けて、x座標を2つ進める必要があります。
この判定はmattnさんの[go-runewidth](https://github.com/mattn/go-runewidth)を使うのが普通です。

そのため、日本語などを表示する場合は、以下のように変更する必要があります。

```go
import (
  runewidth "github.com/mattn/go-runewidth" // 追加
)

var str []rune = []rune("こんにちは世界！")
var x, i = 0, 0

// その他のイベント
func otherEvent(screen tcell.Screen) {
	if i >= len(str) {
		x = 0
		i = 0
		screen.Clear()
	}
	screen.SetContent(x, 0, str[i], nil, tcell.StyleDefault)
	i++
	x += runewidth.RuneWidth(str[i])
}
```

文字(rune)のカウントとx座標のカウントを別に行っています。

### あいまい幅

上記で、多くの文字は対応できていますが、注意すべき点があります。
Unicodeで、一つのコードで幅が1でも2でも良い文字があります。これをあいまい幅と言います。
ターミナルエミュレーターによって、あいまい幅の幅が決定されますが、アプリケーション側から、その幅を得る統一的な方法がありません。

ターミナルエミュレーターによっては、設定できる場合があります。

![tcell-ambiguous.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/18555/b6a363c0-1fae-6fc4-2e44-f4eee13b1c43.png)

曖昧幅の文字(w)： 半角 or 全角

この幅とrunewidth.RuneWidthが返す幅が異なる場合があります。そうすると文字が重なって表示されなくなったり、空白ができたりします。

例えば、"┌───────こんにちは世界！───────┐"のような文字列を表示したい場合があります。この場合、罫線があいまい幅であるため、ターミナルエミュレーターによって表示が変わってしまいます。

そのため、`RUNEWIDTH_EASTASIAN`という環境変数を設定して、あいまい幅の幅を指定することが必要な場合があります。

```go
var str []rune = []rune("┌───────こんにちは世界！───────┐")
var x, i = 0, 0

// その他のイベント
func otherEvent(screen tcell.Screen) {
	if i >= len(str) {
		x = 0
		i = 0
		screen.Clear()
	}
	screen.SetContent(x, 0, str[i], nil, tcell.StyleDefault)

	w := fmt.Sprintf("%d", runewidth.RuneWidth(str[i]))
	screen.SetContent(x, 1, []rune(w)[0], nil, tcell.StyleDefault)

	x += runewidth.RuneWidth(str[i])
	i++
}
```

* 曖昧幅が半角、`RUNEWIDTH_EASTASIAN=0`の場合(A)
![A](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/18555/ca161d84-0d30-469a-6d2a-ce5e635f397f.png)
* 曖昧幅が全角、`RUNEWIDTH_EASTASIAN=1`の場合(B)
![B](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/18555/bfd5abd7-02d5-c305-1116-36ecbc2ee768.png)
* 曖昧幅が全角、`RUNEWIDTH_EASTASIAN=0`の場合(C)
![C](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/18555/321004c8-a0bc-19b8-ab4c-b676ee7c895a.png)
* 曖昧幅が半角、`RUNEWIDTH_EASTASIAN=1`の場合(D)
![D](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/18555/4cea150e-9bb6-e313-934e-ab194352a838.png)

A,Bはあっていますが、C,Dはあっていないため、表示に問題がでています。
ターミナルエミュレーターに合わせて環境変数を設定してもらえば良いですが、あいまい幅を仮定してアプレケーションを構築してしまうと、ターミナルエミュレーターでは設定できなかったり、設定してもらうのが難しい場合があるので注意が必要です。

### 結合文字

Goのruneは、厳密には1文でではなく、2つ以上のruneを組み合わせて1つの文字を表現することができます。これを結合文字と言います。tcellでは、結合文字をサポートしています。

例えば、"👨‍👩‍👧‍👦"のような絵文字は、"👨"と"👩"と"👧"と"👦"を組み合わせて1つの文字として表示しています。

