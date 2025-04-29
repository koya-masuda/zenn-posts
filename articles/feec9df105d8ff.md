---
title: "【Ruby】M1macでmysql2がインストールできないとき"
emoji: "🔑"
type: "tech"
topics:
  - "mysql"
  - "rails"
  - "ruby"
  - "mac"
  - "m1"
published: true
published_at: "2022-03-14 14:53"
---

# 背景
RubyのORマッパー、mysql2がbundle installできなかった。

# 環境

* MacBook Air (M1, 2020)
* macOS Big Sur **11.2.3**
* cpu : Apple M1 (**arm64**)
* ruby **2.7.2p137** (2020-10-01 revision 5445e04352) [arm64-darwin20]
* Bundler version **2.2.9**
* rbenv **1.1.2**

# エラー詳細

`bundle install`を実行したところ、以下のエラーメッセージが出た。

```plaintext

~~~~~~（省略）~~~~~~
2 warnings generated.
compiling infile.c
compiling mysql2_ext.c
compiling result.c
compiling statement.c
linking shared-object mysql2/mysql2.bundle
ld: warning: directory not found for option '-L/usr/local/opt/openssl/lib'
ld: library not found for -lssl
clang: error: linker command failed with exit code 1 (use -v to see invocation)
make: *** [mysql2.bundle] Error 1

make failed, exit code 2

Gem files will remain installed in
/Users/パス/.bundle/ruby/2.7.0/gems/mysql2-0.5.3
for inspection.
Results logged to
/Users/root/Documents/パス/.bundle/ruby/2.7.0/extensions/arm64-darwin-20/2.7.0/mysql2-0.5.3/gem_make.out

An error occurred while installing mysql2 (0.5.3), and Bundler cannot
continue.
Make sure that `gem install mysql2 -v '0.5.3' --source 'https://rubygems.org/'`
succeeds before bundling.

In Gemfile:
  mysql2

```


# 調査



先のエラーメッセージに

```
ld: warning: directory not found for option '-L/usr/local/opt/openssl/lib'
ld: library not found for -lssl
```
と書かれています。

`libssl.dylib`を探しに行くパスは`/usr/local/opt/openssl/lib`となっています。
`/usr/local/opt/openssl/lib`こんなディレクトリあるのか??

(`ld`や`-lssl`に関しては、以下の記事に詳しく書かれています。
[mysql2 gemインストール時のトラブルシュート](https://qiita.com/HrsUed/items/ca2e0aee6a2402571cf6))

`ls`で実際に見てみると

```
$ ls /usr/local/opt/openssl/lib
ls: /usr/local/opt/openssl/lib: No such file or directory
```
ありませんでした。

ありもしないパスを指定していたのが原因と判断しました。

# 解決法
今回、用いるopensslはどこにあるのか。
`$ brew info openssl`で確認できる

```plaintext
$ brew info openssl
openssl@1.1: stable 1.1.1k (bottled) [keg-only]
Cryptography and SSL/TLS Toolkit
https://openssl.org/
/opt/homebrew/Cellar/openssl@1.1/1.1.1k (8,071 files, 18MB)
  Poured from bottle on 2021-04-01 at 15:21:54
From: https://github.com/Homebrew/homebrew-core/blob/HEAD/Formula/openssl@1.1.rb
License: OpenSSL
==> Caveats
A CA file has been bootstrapped using certificates from the system
keychain. To add additional certificates, place .pem files in
  /opt/homebrew/etc/openssl@1.1/certs

and run
  /opt/homebrew/opt/openssl@1.1/bin/c_rehash

openssl@1.1 is keg-only, which means it was not symlinked into /opt/homebrew,
because macOS provides LibreSSL.

If you need to have openssl@1.1 first in your PATH, run:
  echo 'export PATH="/opt/homebrew/opt/openssl@1.1/bin:$PATH"' >> ~/.zshrc

For compilers to find openssl@1.1 you may need to set:
  export LDFLAGS="-L/opt/homebrew/opt/openssl@1.1/lib"
  export CPPFLAGS="-I/opt/homebrew/opt/openssl@1.1/include"

For pkg-config to find openssl@1.1 you may need to set:
  export PKG_CONFIG_PATH="/opt/homebrew/opt/openssl@1.1/lib/pkgconfig"

==> Analytics
install: 906,686 (30 days), 2,526,355 (90 days), 8,960,513 (365 days)
install-on-request: 75,261 (30 days), 306,361 (90 days), 1,237,150 (365 days)
build-error: 0 (30 days)
```

このメッセージのこの部分

```
For compilers to find openssl@1.1 you may need to set:
  export LDFLAGS="-L/opt/homebrew/opt/openssl@1.1/lib"
  export CPPFLAGS="-I/opt/homebrew/opt/openssl@1.1/include"
```

ありました。
`LDFLAGS="-L/opt/homebrew/opt/openssl@1.1/lib"`を
`gem install`の`build--flags`オプションに渡せば解決しそうです。



#### bundlerオプションの設定
http://ruby.studio-kingdom.com/bundler/bundle_config/
↑ここを参照すると、bundler側でgemのインストールにおける、buildオプションを設定できるとのこと。
`gem install`のオプションでも可能ですが、bundlerでgemを管理しているので、こちらのほうが良さそうです。


```(2021.09.30追記)
以下でcppflagsとldflagsの2つをオプションで設定していますが、
1. ldflags
2. cfflags
↑の順に1で解決しなかったら2を試すと解決する場合もあるようです、、、
```

```
$ bundle config --local build.mysql2 "--with-ldflags=-L/パス/lib"
```
↑これで、bundlerにオプション設定を行います。`--local`でプロジェクト毎にbundlerオプションを管理できます。
ここに、さっき確認したopensslのパスを渡して、

```
$ bundle config --local build.mysql2 "--with-ldflags=-L/opt/homebrew/opt/openssl@1.1/lib"
```


再度、`bundle install`

```
$ bundle install
```

できました!!!

```
Fetching mysql2 0.5.3
Installing mysql2 0.5.3 with native extensions

Bundle complete! ** Gemfile dependencies, ** gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
```

### まとめ
1. `brew info openssl`でopensslの場所を確認
1. bundlerのオプションに1で入手したパスを渡す

以上で解決しました


## 原因

bundlerのオプションにopensslのパスを渡せていなかった。



# 参考
[【Rails】MySQL2がbundle installできない時の対応方法
](https://qiita.com/fukuda_fu/items/463a39406ce713396403)


