---
title: efm-langserverの設定について今更ながらやってみた(Go言語)
date: 2019-11-19 18:35:07
tags:
- vim
- LSP
---

[efm-langserver](https://github.com/mattn/efm-langserver)が出たときにめっちゃ便利そうやんーと思っていたんですが、色々ハマったりして時間もなかったのでしばらく放置していましたが、ついに設定してみました。

### efm-langserverとは

> General purpose Language Server that can use specified error message format generated from specified command. This is useful for editing code with linter.

efm-langserverは特定のコマンドから生成されるエラーメッセージに特化したLanguage Serverです。
例えばlinterとかが例に入ります。

### 導入例

efm-langserverを導入する利点は以下のような点が挙げられると思います。

- 言語のLSが存在しないが、Linterは存在する場合
- LSはあるが、Linterを使いたい場合

1つ目の利用方法はefm-langserverのREADMEを見てもらえば出来ると思うので、今回は下について考えながら設定していきたいと思います。

### なぜefm-langserverを利用するか？

linterのエラーを出力するプラグインとしては[ALE](https://github.com/dense-analysis/ale)が有名だと思います。
ALEはすごく便利なプラグインで、Linterさえ導入すればVimにエラー文などを出力してくれます。
しかし、便利な反面プラグインとして大きすぎるという欠点があります。

そこで今回は[vim-lsp](https://github.com/prabirshrestha/vim-lsp)だけで全てが完結するような設定を目指します。

### 早速設定

今回はGo言語の利用例について考えながらやります。
Go言語では[gopls](https://github.com/golang/tools/tree/master/gopls)というLanguage Serverを使います。
また、efm-langserverで使うlinterとして[golint](https://github.com/golang/lint)を使用します。

早速設定

```vim
if executable('gopls')
    au User lsp_setup call lsp#register_server({
        \ 'name': 'go',
        \ 'cmd': {server_info->['gopls']},
        \ 'whitelist': ['go'],
        \ 'workspace_config': {'gopls': {
        \     'usePlaceholders': v:true,
        \     'completeUnimported': v:true,
        \   }},
        \ })
endif

if executable('efm-langserver')
  augroup LspEFM
    au!
    autocmd User lsp_setup call lsp#register_server({
        \ 'name': 'efm-langserver',
        \ 'cmd': {server_info->['efm-langserver', '-c='.$HOME.'/.config/efm-langserver/config.yaml', '-log='.g:log_files_dir.'/efm-langserver.log']},
        \ 'whitelist': ['go'],
        \ })
  augroup END
endif
```

1つ目がgoplsの設定で2つ目がefm-langserverの設定です。

そして、`~/.config/efm-langserver/config.yaml`に以下のような設定を行います。

```yaml
languages:
  go:
    lint-command: 'golint -set_exit_status=1'
    lint-formats:
      - '%f:%l:%c:%m'
```

これで両立可能です！
以下の画像が実際の様子です。
関数の`Hello`と`hello`がタイポしているので起こっているエラーがgoplsのエラーです。
`Hello`という公開関数に対してコメントを書きなさいと怒られているエラーがgolintのエラーです。
ちゃんと設定できてますね！

<img src="/css/images/efm-langserver.png" alt="" align="left" style="max-height: 500px;">
<br style="clear:left;">

これで終わりでもいいのですが、efm-langserverについての補足を少しだけします。

### linterのexitステータスは1である必要がある

efm-langserverはexistステータスに以上がある場合にエラーをlinterの内容を出力する設定になっています。
`golint`は何故かexitステータスがデフォルトだと0になるようになっているみたいなので、`-set_exit_status=1`というように指定する必要があります。

### lint-formatsを設定する

lint-formatsはefm-langserverの設定でデフォルトだと、`%f:%l:%m`と`%f:%l:%c:%m`の2つが設定されています。
この2つでパースするとgolintは両方の設定に成功してしまします。
そして、Diagnosticsが2つ送られて来てしまい、Diagnosticsが２つ存在してしまいます。
なので、`lint-formats:`で正しい方のフォーマットを指定しました。

ちなみに、エラーformatは[Vimのエラーフォーマット構文](https://vim-jp.org/vimdoc-en/quickfix.html#error-file-format)を使っているみたいなので、こちらを参考にしてください。

## まとめ

goでgoplsの他にgolintのエラーも出るようになった！

#### ( 2019/12/06 追記

11月末に取り込まれた[変更](https://github.com/mattn/efm-langserver/commit/0bbc17debeaa88e224817d6364cb8d5b0c8f388f)によってexitステータスが0でも行けるようになりました。
その場合以下のような設定でいけます。

```yaml
languages:
  go:
    lint-command: 'golint'
    lint-ignore-exit-code: true
    lint-formats:
      - '%f:%l:%c:%m'
```

