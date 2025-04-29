---
title: "RubyでChromeを自動操縦してスクレイピング"
emoji: "⛰️"
type: "tech"
topics:
  - "chrome"
  - "ruby"
  - "web"
  - "スクレイピング"
  - "ferrum"
published: true
published_at: "2022-07-28 20:20"
---

# この記事で扱うこと・扱わないこと

## 扱うこと
- RubyでWebDriverを経由しないブラウザ操作

## 扱わないこと
- seleniumを使用したブラウザ操作

# TL;DR

[chrome_remoteのサンプルソース](#chrome_remoteサンプルコード)

[ferrumのサンプルソース](#chrome_remoteサンプルコード)

# 環境

```
Ruby: 2.7.2
```

# ブラウザの自動操縦

## selemiumの辛いところ

ブラウザの自動操縦といえば[selenium](https://www.selenium.dev/ja/documentation/)でしょう。Rubyにも`selenium-webdriver`というgemがあります。ただseleniumの辛い点はChromeのバージョンに沿ったdriverを管理しないといけないこと。ブラウザなんて気がついたらバージョン上がってます。セキュリティの観点もありバージョン固定なんて嫌ですね。

**seleniumを動かすときの構成**
プログラム⇔driver⇔ブラウザ

## Chrome DevTools Protocol

Chromium系ブラウザには開発者用のデバッグモードがあります。このデバッグ機能を使うにはブラウザ起動時にdebugging-portオプションをつける必要があります。このときdebugging-port経由でブラウザを操作するためのプロトコルが[Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/)です。

```txt:Macの例
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222
```

```txt:Windowsの例
C:/Program Files/Google/Chrome/Application/chrome.exe --remote-debugging-port=9222
```

# RubyでCDPを扱う

RubyでCDPを扱うには[chrome_remote](https://github.com/cavalle/chrome_remote)というライブラリがありますが、READMEの冒頭で
>This library is not actively maintained. Check https://github.com/rubycdp/ferrum for a replacement.

とあります。それじゃあ使ってみるか、**ferrum**

## chrome_remoteのおさらい

RubyのプログラムとChromeは切り離されているので、debugging-port付きでchromeが起動していることが前提となります。
9222でリッスンしているかは以下のコマンドで確認しましょう。debigging-portオプション付きで起動できていたら、タブの情報がjsonで返ってきます。

```curl
curl localhost:9222/json
```

### chrome_remoteサンプルコード

今回はYahooのtitleを取ってきています。

```rb:chrome_remote
require 'chrome_remote'
browser = ChromeRemote.client
browser.send_cmd('Page.navigate', url: 'https://www.yahoo.co.jp/')
browser.send_cmd('Runtime.evaluate', expression: "document.querySelector('title').textContent")['result']['value']
# {"result"=>{"type"=>"string", "value"=>"Yahoo! JAPAN"}}
# => "Yahoo! JAPAN"
```

JavaScriptを実行したいときには[Runtime#evaluate](https://chromedevtools.github.io/devtools-protocol/tot/Runtime/#method-evaluate)でRubyのHashが返ってきます。

# ferrumとは

[ferrum](https://github.com/rubycdp/ferrum)とは`chrome_remote`同様にCDPによるChromeの操作をRubyで扱えるようにしたライブラリです。`chrome_remote`では何でもかんでも`send_cmd`メソッドから自前でCDPのリファレンスを調べて、実装する必要がありました。一方で、`ferrum`では自動操縦やスクレイピングに必要なメソッドはきれいにラップして用意してくれてます。

## ferrumサンプルコード

`ferrum`が`chrome_remote`と違うのはブラウザの立ち上げから行うことです。現状対応しているブラウザはChromeとFirefoxです。使いたいブラウザがあったり、ChromeやFirefoxなのにブラウザが立ち上がらない！ってなったら[ココ](https://github.com/rubycdp/ferrum/blob/main/lib/ferrum/browser/options/chrome.rb)にパッチを当ててください。
>The Chrome DevTools Protocol allows for tools to instrument, inspect, debug and profile Chromium, Chrome and other Blink-based browsers. 

ただし、レンダリングエンジンにBlinkが使われていることが条件かも？？ただFirefoxのレンダリングエンジンGeckoだし、、ここはあまり詳しくないです、、
2行目のnewしたときにブラウザインスタンスが生成されます。デフォルトでヘッドレスモードなので、今回は`headless: false`としています。

```rb:ferrum
require 'ferrum'
browser = Ferrum::Browser.new(headless: false, port: 9222)
browser.go_to('https://www.yahoo.co.jp/')
browser.evaluate("document.querySelector('title').textContent")
browser.quit
# => "Yahoo! JAPAN"
```

スクレイピングなどで用いる基本的なサンプルコードを紹介します。

## CDPでスクロール

### Ferrum::Mouse#Ferrum::Mouse#scroll_to

自動のE2Eテストやスクレイピングしていたらスクロールで発火するJSを待ちたいときがあります。
`ferrum`にちょうどいいメソッドを発見。メソッド名もわかりやすくていいなーと思いましたが、ソースを覗いてみるとJSで`window.scrollTo`しているだけ（泣）
せっかくラップするならCDPでやりきってほしかった。

```rb:github.com/rubycdp/ferrum/blob/main/lib/ferrum/mouse.rb
module Ferrum
  class Mouse
    ~~省略~~
    def scroll_to(top, left)
      tap { @page.execute("window.scrollTo(#{top}, #{left})") }
    end
    ~~省略~~
  end
end
```

### ferrumでCDPを直に使う

`Ferrum::Page#command`を使います。使い方は`chrome_remote`のsend_cmdと同じです。
定義元はココです↓
https://github.com/rubycdp/ferrum/blob/024ad04d4f052137c6cd4eb3f54be32f737c0b1e/lib/ferrum/browser/client.rb

### ferrumでCDP直書きスクロール

`Input.dispatchMouseEvent`を使います。`type: mouseWheel`を選択。

(x, y) = (0, 0)から画面下方向へ500pxスクロールするには

```rb
browser.page.command(
  'Input.dispatchMouseEvent',
  type: 'mouseWheel',
  x: 0,
  y: 0,
  deltaX: 0, deltaY: 500
)
```

連続して3秒おきに500pxずつ3回スクロールしたかったら、

```rb
3.times do |i|
  browser.page.command(
    'Input.dispatchMouseEvent',
    type: 'mouseWheel',
    x: 0,
    y: 500 * (i - 1),
    deltaY: 500 * i
  )
  sleep(3)
end
```

でスクロールを再現できます。

## ページ遷移

ferrumなら

```rb
browser.go_to('https://github.com/rubycdp/ferrum')
```

素のCDPならこう書く

```rb
browser.page.command('Page.navigate', url: 'https://github.com/rubycdp/ferrum')
```
 
## JS実行

`ferrum`には大きく分けて2つのJS実行メソッドが用意されています。

1. evaluate
返り値がほしいときに使う

```js
例)
browser.evaluate("document.querySelector('title').textContent;")
```

2. execute
返り値がいらないときに使う

```js
browser.execute("window.scrollBy(0, window.innerHeight);")
```

これくらいできたら十分にでしょう。JSだけで完結するもよし、HTMLを[Nokogiri](https://nokogiri.org/)に食わせてもよし。
これを読んだあなたに快適なスクレイピング生活が訪れますように。

# 参考

- https://chromedevtools.github.io/devtools-protocol/
- https://github.com/rubycdp/ferrum
- https://github.com/cavalle/chrome_remote
- https://engineering.mercari.com/blog/entry/20220128-3a0922eaa4/