WindowsにDockerをインストールして開発を行っております。

現在作っているものとは別に環境を作る必要が出たために、もうひとつ別のディレクトリを/home/[user_name]/配下に配置してVSCodeで開こうとすると、何やらPermission DeninedによりWSLに接続できない問題が発生。

```
sh: 1: /mnt/c/Users/user/.vscode/extensions/ms-vscode-remote.remote-wsl-0.44.4/scripts/wslServer.sh: Permission denied
```
VSCodeからWSLにつなごうとすると`VS Code Server for WSL closed unexpectedly`が表示されて失敗する問題は、検索すると同じような事例がヒットします。
しかし問題の原因そのものは異なるようで、例えば[こちらの記事](https://ginpen.com/2021/09/25/vs-code-server-fow-swl-closed-unexpectedly/)の内容を試しても解決はしませんでした。そもそもエラーメッセージが異なっているので必要な対処も違って当然ですが…

## wslServer.sh: Permission denied で検索
こちらのstackoverflowのページがヒットし、書かれていた回答で解決しました。

https://stackoverflow.com/questions/62644034/permission-issue-with-vscode-and-wsl-code-command-not-found

> WSL初心者で、Windows10にVS codeをインストーラーでネイティブインストールしました。現在、WSLからVSコードのコマンドコードを実行しようとすると、以下のメッセージが表示されます。

```
bash: /mnt/c/Users/user/AppData/Local/Programs/Microsoft VS Code/bin/code: Permission denied
```

> また、VS Code Server for WSLを実行しようとすると、同様のメッセージが表示されます。

```
[2020-06-29 17:41:41.640] Launching C:\windows\System32\wsl.exe -d Ubuntu sh -c '"$VSCODE_WSL_EXT_LOCATION/scripts/wslServer.sh" cd9ea6488829f560dc949a8b2fb789f3cdc05f5d stable .vscode-server 0  ' in c:\Users\user\.vscode\extensions\ms-vscode-remote.remote-wsl-0.44.4}
[2020-06-29 17:41:41.779] sh: 1: /mnt/c/Users/user/.vscode/extensions/ms-vscode-remote.remote-wsl-0.44.4/scripts/wslServer.sh: Permission denied
[2020-06-29 17:41:41.780] VS Code Server for WSL closed unexpectedly.
```

ドンピシャおんなじ問題です。

末尾に投稿されていた回答がこちら。

> それは、あなたの/etc/wsl.confに関係しています。ファイルの内容を次のように変更します

```
[automount]
enabled=true
root = /
options="metadata,uid=1000,gid=1000,umask=002,dmask=002,fmask=002"
```
ほうほう、最初に「WSL起動時、ついでにDockerも起動する」という設定を書き込んだ時には全く気にしなかったこの部分が関係していたとは。

「この設定の意味はここで詳しく解説されてるよ」ということだったので、リンク先を読んでみます。

https://askubuntu.com/questions/429848/dmask-and-fmask-mount-options#:%7E:text=fmask%20and%20dmask%20are%20mount,files%20and%20dmask%20to%20directories

Ask Ubuntuという、UbuntuOSを対象とした質問サイトのようですね。そんなものがあったとは。

## "dmask" and "fmask" mount options

リンク先はこんな質問でした。

> このコマンドで手動でマウントしてみました。

```
sudo mount -t vfat /dev/sdb1 /media/external -o uid=1000,gid=1000,utf8,dmask=027,fmask=137
```

> dmaskとfmaskが何をするものなのかがよくわかりません。パーミッションの設定に使われるのは知っていますが、マウントされたディレクトリ内のファイルやフォルダのパーミッションを確認すると、fmaskやdmaskを使って設定したものと違っています。

> では、実際に何をしているのでしょうか？

### 回答

> fmask と dmask は FAT ファイルシステムのマウントオプションで、fstab に基づいています。

> これらはパーミッションを定義するために使われます (umask はファイルとディレクトリの両方に設定しますが、fmask はファイルだけに、dmask はディレクトリに適用されます)。

> マスクはファイルのパーミッションではなく、あなたが望むパーミッションを得るために使用されます。さらに、マスクはいかなるパーミッションも追加することはできず、ファイルやディレクトリが持つことのできるパーミッションを制限するだけです。

> umaskは、ファイルやフォルダのデフォルトです、あなたがファイルやフォルダのパーミッションをカスタマイズしたい場合は、umaskと同じように使用するfmaskとdmaskを使用する必要があります。

> マスクのパーミッションは、しかし、この表は本当にマスクのパーミッションがどのように動作するかを理解するのに便利です、chmodコマンドに渡された8進数のパーミッションコードのようなものではありません。

```
    0   1   2   3   4   5   6   7
r   +   +   +   +   -   -   -   -
w   +   +   -   -   +   +   -   -
x   +   -   +   -   +   -   +   -
```

> 例えば、パーミッションを 0777 に設定したい場合は、umask に 0000 を設定する必要があり（例：umask=0000）、0755 に設定したい場合は、0022 を設定することになります。

> 最初の文字は、8進数の許可証であることを表します。
> 2番目は所有者
> 3番目はグループ
> 4番目はその他、つまり他のユーザーです。

## なんのことやら

DeepL翻訳の自然な日本語でもなんのことやらまるでわかりませんけど、パーミッションの例を見るに、例えば今回`0002`に設定した部分に関しては`chmod 775`を実行したのと同じということなんですかね。

https://qiita.com/yummy888/items/510f82976780151855f8

こちらの記事を読むに「その他」への実行権限を与えたということなんじゃないかな…

Linuxの知識が全然ないのでさっぱりですが、そのへんわかったとて、今までは普通にVSCodeとRemote-Containerで開発できていた(エラーが出ていた間も、開発中ディレクトリに関しては問題なく読み書きできた)のがなぜなのかはわからないような気もする……疲れたのでいったん開発に頭を切り替えます。

とりあえず調べたこと・見たことを備忘録としてまとめました。