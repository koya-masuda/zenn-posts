---
title: "初めてOSSにPRを投げた話"
emoji: "🍣"
type: "tech"
topics:
  - "ruby"
  - "graphql"
  - "oss"
published: true
---

# 経緯

RubyKaigi 2025に参加し、#kaigieffectとしてOSS活動を始めたいと考えていました。業務でGraphQLを使用していることもあり、[graphql-ruby](https://github.com/rmosolgo/graphql-ruby)に触れてみることに決めました。どうせならRubyの最新（Head）バージョンで開発しようとセットアップを進めていたところ、Headでは動作しない箇所を見つけたため、PRを送ることにしました。

# 開発環境のセットアップ

セットアップ手順は以下の通りです。

> Get your own copy of graphql-ruby by forking rmosolgo/graphql-ruby on GitHub and cloning your fork.
>
> Then, install the dependencies:
>
> - Install SQLite3 and MongoDB (eg, brew install sqlite && brew tap mongodb/brew && brew install mongodb-community)
> - bundle install
> - rake compile # If you get warnings at this step, you can ignore them.
> - Optional: Ragel is required to build the lexer

https://graphql-ruby.org/development

## ローカル環境

```
❯ ruby -v
ruby 3.5.0dev (2025-01-18T00:19:17Z master 65a7c69188) +PRISM [arm64-darwin23]

❯ bundle -v
Bundler version 2.7.0.dev
```

## bisonのバージョンが低い

`bundle install`後の`rake compile`にてエラーが発生しました。bisonのバージョンが低いことが原因のようです。


```
❯ rake compile
cd tmp/arm64-darwin23/graphql_c_parser_ext/3.5.0
/opt/homebrew/bin/gmake
yacc  ../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y
../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y:1.10-14: require bison 3.8, but have 2.3
gmake: *** [<builtin>: parser.c] Error 63
rake aborted!
Command failed with status (2): [/opt/homebrew/bin/gmake]

Tasks: TOP => compile => compile:arm64-darwin23 => compile:graphql_c_parser_ext:arm64-darwin23 => copy:graphql_c_parser_ext:arm64-darwin23:3.5.0 => tmp/arm64-darwin23/graphql_c_parser_ext/3.5.0/graphql_c_parser_ext.bundle
(See full trace by running task with --trace)
```

bisonを新しくインストールし、PATHを通して、シェルを再起動して対応します。

```
brew install bison
echo 'export PATH="/opt/homebrew/opt/bison/bin:$PATH"' >> ~/.zsh_profile
exec $SHELL -l
```

bisonのバージョンが更新されました👏

```
❯ bison --version
bison (GNU Bison) 3.8.2
Written by Robert Corbett and Richard Stallman.

Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

`rake compile`のリトライ！

```
❯ rake compile
cd tmp/arm64-darwin23/graphql_c_parser_ext/3.5.0
/opt/homebrew/bin/gmake
yacc  ../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y
../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y:1.1-8: warning: POSIX Yacc does not support %require [-Wyacc]
    1 | %require "3.8"
      | ^~~~~~~~
../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y:1.10-14: warning: POSIX Yacc does not support string literals [-Wyacc]
    1 | %require "3.8"
      |          ^~~~~
../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y:2.1-7: warning: POSIX Yacc does not support %define [-Wyacc]
    2 | %define api.pure full
      | ^~~~~~~
../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y:3.1-7: warning: POSIX Yacc does not support %define [-Wyacc]
    3 | %define parse.error detailed
      | ^~~~~~~
../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y: warning: 19 shift/reduce conflicts [-Wconflicts-sr]
../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y: warning: 62 reduce/reduce conflicts [-Wconflicts-rr]
../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y: note: rerun with option '-Wcounterexamples' to generate conflict counterexamples
mv -f y.tab.c parser.c
compiling parser.c
y.tab.c:1669:9: warning: variable 'yynerrs' set but not used [-Wunused-but-set-variable]
 1669 |     int yynerrs = 0;
      |         ^
../../../../graphql-c_parser/ext/graphql_c_parser_ext/parser.y:896:61: warning: function 'yyerror' could be declared with attribute 'noreturn' [-Wmissing-noreturn]
  896 | void yyerror(VALUE parser, VALUE filename, const char *msg) {
      |                                                             ^
2 warnings generated.
linking shared-object graphql/graphql_c_parser_ext.bundle
cd -
mkdir -p tmp/arm64-darwin23/stage/graphql-c_parser/lib/graphql
/opt/homebrew/bin/gmake install sitearchdir=../../../../graphql-c_parser/lib/graphql sitelibdir=../../../../graphql-c_parser/lib/graphql target_prefix=
/opt/homebrew/bin/ginstall -c -m 0755 graphql_c_parser_ext.bundle ../../../../graphql-c_parser/lib/graphql
cp tmp/arm64-darwin23/graphql_c_parser_ext/3.5.0/graphql_c_parser_ext.bundle tmp/arm64-darwin23/stage/graphql-c_parser/lib/graphql/graphql_c_parser_ext.bundle
```

むむ〜。なんかゴニョゴニョ出てる。けど、warningは無視していいとのことなので無視。

> rake compile # If you get warnings at this step, you can ignore them.

# Unit Testsの実行

環境のセットアップができたらテストを実行してみます。

```
bundle exec rake test   # tests only
```
https://graphql-ruby.org/development#running-the-tests

```
❯ bundle exec rake test
/Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/3.5.0+0/readline.rb:4: warning: reline was loaded from the standard library, but is not part of the default gems starting from Ruby 3.5.0.
You can add reline to your Gemfile or gemspec to silence this warning.
/Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/gems/3.5.0+0/gems/byebug-12.0.0/lib/byebug/commands/irb.rb:4: warning: irb was loaded from the standard library, but is not part of the default gems starting from Ruby 3.5.0.
You can add irb to your Gemfile or gemspec to silence this warning.
/Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/gems/3.5.0+0/gems/libev_scheduler-0.2/lib/libev_scheduler.rb:16: warning: method redefined; discarding old process_wait
/Users/koyamasuda/personal/graphql-ruby/spec/spec_helper.rb:47: warning: benchmark was loaded from the standard library, but is not part of the default gems starting from Ruby 3.5.0.
You can add benchmark to your Gemfile or gemspec to silence this warning.
Coverage report generated for RSpec to /Users/koyamasuda/personal/graphql-ruby/coverage.
Line Coverage: 52.43% (108 / 206)
Branch Coverage: 8.7% (2 / 23)
Lcov style coverage report generated for RSpec to /Users/koyamasuda/personal/graphql-ruby/coverage/lcov/graphql-ruby.lcov
Stopped processing SimpleCov as a previous error not related to SimpleCov has been detected
/Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/3.5.0+0/bundled_gems.rb:76:in 'Kernel.require': cannot load such file -- benchmark (LoadError)
Did you mean?  benchmark/ips
        from /Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/3.5.0+0/bundled_gems.rb:76:in 'block (2 levels) in Kernel#replace_require'
        from /Users/koyamasuda/personal/graphql-ruby/spec/spec_helper.rb:47:in '<top (required)>'
        from /Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/3.5.0+0/bundled_gems.rb:76:in 'Kernel.require'
        from /Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/3.5.0+0/bundled_gems.rb:76:in 'block (2 levels) in Kernel#replace_require'
        from /Users/koyamasuda/personal/graphql-ruby/spec/graphql/analysis/field_usage_spec.rb:2:in '<top (required)>'
        from /Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/3.5.0+0/bundled_gems.rb:76:in 'Kernel.require'
        from /Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/3.5.0+0/bundled_gems.rb:76:in 'block (2 levels) in Kernel#replace_require'
        from /Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/gems/3.5.0+0/gems/rake-13.2.1/lib/rake/rake_test_loader.rb:21:in 'block in <main>'
        from /Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/gems/3.5.0+0/gems/rake-13.2.1/lib/rake/rake_test_loader.rb:6:in 'Array#select'
        from /Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/gems/3.5.0+0/gems/rake-13.2.1/lib/rake/rake_test_loader.rb:6:in '<main>'
rake aborted!
Command failed with status (1)
/Users/koyamasuda/.rbenv/versions/ruby-dev/bin/bundle:25:in 'Kernel#load'
/Users/koyamasuda/.rbenv/versions/ruby-dev/bin/bundle:25:in '<main>'
Tasks: TOP => test
(See full trace by running task with --trace)
```

おっ、落ちたぞ👀（OSS活動目的だったのでちょっと嬉しかったですw）
エラーメッセージを要約すると「`require 'benchmark'`を実行しようとした際に、benchmarkライブラリが見つからず読み込めなかった」ようです。

```
/Users/koyamasuda/.rbenv/versions/ruby-dev/lib/ruby/3.5.0+0/bundled_gems.rb:76:in 'Kernel.require': cannot load such file -- benchmark (LoadError)
Did you mean?  benchmark/ips
```

## エラーの原因と修正

```
/Users/koyamasuda/personal/graphql-ruby/spec/spec_helper.rb:47: warning: benchmark was loaded from the standard library, but is not part of the default gems starting from Ruby 3.5.0.
You can add benchmark to your Gemfile or gemspec to silence this warning.
```

テスト時のwarningを見ると、Ruby3.5からは`benchmark`が標準ライブラリから外れたみたいです。
エラーメッセージの通りに`require "benchmark/ips"`と直すと、`ostruct`gemでも同じことが起きているようでした。ただ、`ostruct`では既に同じ対応をしている[PR](https://github.com/rmosolgo/graphql-ruby/pull/5208)がcloseしていたので、対応は`benchmark`のみに留めました。

```diff
- require "benchmark"
+ require "benchmark/ips"
```

## Pull requestの作成

そのままPR（Pull request）を作成しました。PRの説明には背景や環境、`ostruct`の対応は敢えてしなかったことなどをなるべく丁寧に記述しました。

実際のPR
https://github.com/rmosolgo/graphql-ruby/pull/5341

## Pull requestのその後

> Hey, thanks for reporting this! When I saw require "benchmark" in the tests, something seemed strange to me ... I don't think the tests use Benchmark. So I removed it and found the tests ran fine.
>
> I added a CI build on head in [#5342](https://github.com/rmosolgo/graphql-ruby/pull/5342), and removed this line in that PR.

テストで`benchmark`を使っていない想定だったようで、メンテナーの方が先回りして直してくださいました。せっかくならコミュニケーション取りながら自分がcommitしたかった！ですが、初めてだったのでこれくらいで丁度良かったと思うことにします。

# まとめ

GitHub flowが初めてだったので、forkしてupsteramの設定など勉強になりました。Headで動かしてみるとissueが見つかりやすいかもしれないですね。

*   **GitHub Flow:** Forkして、ローカルで修正して、upstreamを設定して...という一連の流れを実際に体験できました
    ```bash
    # upstreamリモートを追加 (初回のみ)
    git remote add upstream https://github.com/rmosolgo/graphql-ruby.git
    # 最新の変更を取り込む
    git fetch upstream
    git merge upstream/master
    ```
*   **Ruby Headバージョン:** 最新の開発版を使うことで、ライブラリの潜在的な問題を発見しやすいかもしれないと感じました
*   **OSSコミュニケーション:** 拙い英語でも、丁寧に説明すればメンテナーの方がちゃんと読んでくれることが分かりました（今回は先回りされちゃいましたが！）

https://x.com/koxya/status/1917008082636574801

初めてのOSSへのPRはドキドキしましたが、やってみると意外ハードルは低いのかなと思いました。
この記事が、これからOSS活動を始めたいと思っている方の、小さな一歩に繋がれば嬉しいです。
