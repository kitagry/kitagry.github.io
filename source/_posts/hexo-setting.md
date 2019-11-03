---
title: hexoをGitHub Pagesで使うための設定
date: 2019-11-03 17:15:43
tags:
- hexo
- Github Pages
- travis
---

せっかくなのでブログを1つくらい更新しようかなと思います。

現在僕はGitHub Pagesとhexoを使ってページを公開しています。
まあ、探せばやり方はいくらでもあると思うのですが、意外と検索しても出てこなかったので僕の設定の備忘録として残しておきます。

### hexoとは？

[hexo](https://hexo.io)とは静的サイトジェネレーターです。
基本的にはマークダウンファイルを書くだけでブログを更新できます。
また、テーマなども複数ありかなり応用が効きそうです。
(僕自身は現在hexo3日目なので、あまり良くわかっていませんが。。)

### Github Pagesについて

[Github Pages](https://pages.github.com/)とはGithubにプッシュするだけでウェブサイトを表示してくれるサービスです。
無料でWebページが作成出来るのでかなり便利です。
ただし、ページを公開するための制約が何個かあります。

1. 静的ファイルしか公開することができない
1. masterブランチまたはgh-pagesブランチにあるファイルを公開する

つまり、静的ファイルを公開する場合には、サーバーやドメインを借りるという手間を削減出来るのでかなり良いと思います。

## デプロイをするときの問題点

実はhexoではマークダウンで書いた後に、`hexo generate`と実行することで静的ファイルを子ディレクトリの中に作成します。そして、多くのGithub Pagesはmasterブランチの `/docs` 以下をウェブサイトとして公開することができます。つまり、hexoの `_config.yaml` を以下の用に書き換えればいいわけです。

```yaml
# Directory
source_dir: source
public_dir: docs  # <-- ここをdocsにする
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:
```

それを masterブランチかgh-pagesブランチにプッシュして、設定で変更すればそのまま公開することができます。


しかし、僕のサイトのような`{username}.github.io`というリポジトリでは上の方法が使えません。何故か、先程のようなリポジトリ名ではmasterブランチに直接静的ファイルを置かないとGithub Pagesを使えないらしいのです。

したがって、僕はこの理不尽な仕様に対応するためにちょっとしたコードを書きました。このときに得た知見をブログで共有しようかなと思います。もし、hexoとGithub Pagesで自分のサイトを運用したいなという人がいれば、参考にしていただけると幸いです。


## 解決策を書く前に

実は上の仕様は以下の簡単なステップで回避することができます。しかし以下の方法には少しだけ欠点があります。
`hexo deploy`とはhexoのウェブページをデプロイするための機能です。今回は `_config.yml` の該当箇所を以下のように直せば動かすことができます。

```yaml
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: git@github.com:{your github name}/{your github name}.github.io.git
  branch: master
```

これを書いた後、 `hexo deploy`と入力することで、Githubのmasterブランチに静的ファイルをプッシュしてくれます。この問題点としては、つまりマークダウンファイルなどのhexoに関係する設定ファイルをGitに残すことができないということです。僕としてはこれは結構辛い問題で、hexoの設定ファイルのバックアップが取れないのでもしものときに設定ファイルを引き継ぐことができません。

おそらく賢い人なら「じゃあ、masterブランチにdeployをしておいて、他のブランチでバックアップを取っておけばいいじゃん」ってなると思います。そうです。つまりウェブページを公開する用とソースコードを管理する用のブランチを分けてしまおうというわけです。これなら、バックアップも取れます。しかし、これだとバックアップとデプロイをいちいちやらないと行けなくなります。これは結構面倒でデプロイを忘れたりしてしまうことが頻発してしまいそうです。そこで、僕はバックアップをしたら自動でデプロイをするような仕組みを作成しました。(無料です。)

## ではやっと解決策

今回のやり方は以下の記事を参考にしました。僕の説明が不十分だと感じたら以下の記事を見てみてください。

- [Hexo io: GitHub Pages](https://hexo.io/docs/github-pages.html)
- [Travis-CIの自動デプロイでSSHを実行する方法 [翻訳]](https://qiita.com/koyayashi/items/13d4ac3a4d84d40b4690)

ではやっと本題へ。

今回の方針としては、masterブランチとは別に deployブランチというものを用意してそのブランチへプッシュされたら、Travis CIで `hexo deploy`を実行するという単純なものです。

`hexo deploy`の設定については先程の [`hexo deploy`を使う](#解決策を書く前に)通りに `_config.yml` を変更してください。

### Travis CIの設定

Travis CIの設定については3つのステップがあります。

1. masterブランチにプッシュするための公開鍵の用意
1. travis CIで使うための秘密鍵の用意
1. travis.ymlの設定

これらを順に解説していきます。

#### masterブランチにプッシュするための公開鍵の用意

まずはmasterブランチにプッシュするためのssh keyファイルを用意します。
以下のコマンドを {username}.github.ioディレクトリの下で実行してください。

```
$ ssh-keygen -t rsa -N '' -f deploy_key
```

これを実行することで、 `deploy_key`(秘密鍵)と `deploy_key.pub`(公開鍵)の2つができます。
**これらは間違ってコミットしないように `.gitignore`に登録しておきましょう。**
では、`deploy_key.pub`をGitHubに登録しましょう。

まず、GitHubの{username}.github.ioの設定ページへ行ってください。そこに、 `Deploy Keys`というタブがあると思います。そこへ行って、`add deploy key`というボタンをクリックします。このフォームへ先程の`deploy_key.pub`の中身を登録します。必ず `Allow write access`にチェックをつけてください。

さあ、これで公開鍵の登録は終了しました。
次にTravisの方で秘密鍵を使うための設定を行っていきます。

### Travis CIで使うための秘密鍵の用意

まずはtravisのCLIを導入していきます。

```
$ gem install travis

# Githubアカウントでログインする
$ travis login
```

次に、先程作成した秘密鍵を暗号化し、travisに設定します。

```
$ touch .travis.yml && travis encrypt-file ./deploy_key --add
```

これを実行すると `deploy_key.enc`ファイルと `.travis.yml`が作成されているのが分かると思います。
次に `.travis.yml`ファイルの中を見ると以下のようなコードが挿入されていると思います。

```yaml
before_install:
- openssl aes-256-cbc -K $encrypted_77965d5bdd4d_key -iv $encrypted_77965d5bdd4d_iv
  -in deploy_key.enc -out ./deploy_key -d
```

これで秘密鍵の設定は完了しました。
** `deploy_key.enc`ファイルをGitで管理するということを忘れないでください。**
.encファイルは暗号化されているのでGithubにプッシュしても大丈夫です。
では最後にtravisの設定をしていきましょう。

### travis.ymlの設定

.travis.ymlの設定は以下の通りです。

```yaml
sudo: false
language: node_js
node_js:
- 12
cache: npm
branches:
  only:
  - deploy
before_install:
- openssl aes-256-cbc -K $encrypted_77965d5bdd4d_key -iv $encrypted_77965d5bdd4d_iv
  -in deploy_key.enc -out ./deploy_key -d ### ここは先程のコードです。
- eval "$(ssh-agent -s)"
- chmod 600 ./deploy_key
- echo -e "Host $SERVER_IP_ADDRESS\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config
- ssh-add ./deploy_key
script:
- hexo generate
- hexo deploy
```

これらの簡単な解説をします。
`branches:` はどのブランチでこのtravisを実行するかを設定しています。今回の場合は `deploy`ブランチにプッシュされた場合にのみmasterへデプロイするという宣言を行っています。

次に `before_install` についてです。このコードはsshキーの設定を行っています。最初の `openssl ....`は先程作成した `deploy_key.enc`を復号化するためのコマンドです。それ以下では復号した `deploy_key`をsshキーとして使う設定を行っています。
`before_install`は以下の `script`の前に実行されます。

最後に `script`についてです。といってもこれについては解説する意味もなさそうですね。笑
ただ、hemlにコンパイルを行い、それらをdeployするだけです。


## 自動デプロイしてみる

それではここまでが終わったら、`deploy`ブランチにプッシュしてみましょう。
そうすると [travis](https://travis-ci.com/)のページを確認するとCIが走っているのが確認出来ると思います。
もし、ログインしていない場合は travisにGithubアカウントでログインしてみましょう。

もし、CIがうまくはしればmasterにデプロイが完了しているはずです。
お疲れさまです。


## 最後に

今回は自分がこのブログを構築するためにした設定についてのブログを書いてみました。
もし、読んでいる人の参考になればとても嬉しいです。


## もしバグが起こった場合

Travisで以下のようなエラーが起った場合についてです。

```
iv undefined
The command "openssl aes-256-cbc -K $encrypted_77965d5bdd4d_key -iv $encrypted_77965d5bdd4d_iv -in deploy_key.enc -out ./deploy_key -d" failed and exited with 1 during .
```

これについては[このissue](https://github.com/travis-ci/travis.rb/issues/607)が参考になりました。
僕の場合の解決方法です。

```
$ travis encrypt-file ./deploy_key --pro
```

これを実行したときに出てくる `openssl ...` をtravis.ymlの同じ部分を置き換えて、新しくできた `deploy_key.enc` と一緒にプッシュしてみましょう。
