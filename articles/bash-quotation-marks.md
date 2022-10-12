---
title: "シェルスクリプトでスペースを含む文字列をコマンドにわたす"
emoji: "🛸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [bash, shell]
published: true
---

音源ファイルパスのリストを読み込み、コーデックを取得するシェルスクリプトを作成しました。
ファイル名にスペースが含まれている際に、コマンドを意図した通りに実行することができず困ってしまったので、対処法を残しておきます。

## NG: 読み込んだファイルパスをそのままコマンドにわたす

ファイル名にスペースが入っていると、`$file` がスペースで分割されて複数の引数として認識されてしまう。

```bash
#!/bin/bash

cat filepath.txt | while read file
do
  ffprobe -loglevel quiet -show_streams -print_format json $file
done
```

## NG: 読み込んだファイルパスをエスケープしてコマンドにわたす

ファイルパスをエスケープしてみたが、結局 `$escaped` がスペースで分割されてしまうのは変わらず、複数の引数として認識されてしまっている様子。

```bash
#!/bin/bash

cat filepath.txt | while read file
do
  escaped=printf %q "$file"
  ffprobe -loglevel quiet -show_streams -print_format json $escaped
done
```

ただし、コマンドライン上ではファイル名にスペースを含む場合は、スペースをエスケープすれば正しくコーデックが取得できた。

```bash
ffprobe -loglevel quiet -show_streams -print_format json 11¥ zenn.mp3
```

## OK: 読み込んだファイルパスを引用符を使ってコマンドにわたす

`$file` を `"` で囲むと、スペースが含まれても正しくファイルパスを読み込んでくれた。
これはダブルクォートで囲むと1つの文字列として扱われるため。

```bash
#!/bin/bash

cat filepath.txt | while read file
do
  ffprobe -loglevel quiet -show_streams -print_format json "${file}"
done
```

## その他の引用符

### シングルクォート `'`

変数展開やワイルドカード等の意味のある文字列もただの文字列として扱われる。

```bash
echo '${single}'
# ${single}
```

### バッククォート `` ` ``

バッククォートで囲まれた文字列をコマンドとして実行した結果の標準出力の文字列として扱われる。

```bash
echo `whoami`
# root
```

## 最後に

コマンドラインではファイル名をエスケープして意図した通りに動かすことができていたので、シェルスクリプトでも同じ考えで進めていたら困ってしまいました。
意図した通りに動かすことはできるようになったのですが、コマンドラインとシェルスクリプトとで何が異なっていたのかは宿題事項です。詳しい人がいたら教えてください。
