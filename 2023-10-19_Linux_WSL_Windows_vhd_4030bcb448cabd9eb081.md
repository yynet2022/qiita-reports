<!--
title:   WSL で仮想ハードディスクを追加してみた（速度比較もしてみた）
tags:    Linux,WSL,Windows,vhd,速度比較
id:      4030bcb448cabd9eb081
private: false
-->
# はじめに

　普段から Windows Subsystem for Linux 2 (WSL2) を使っているが、最近ホームのディスク空き容量が気になってきた。

　例えば WSL2 の Linux Kernel をコンパイルしてみたり (約7GB)、Qt6 をデバッグ付きでコンパイルして更に PyQt6 も構築してみたり (約70GB)、メモリが足りないからと一時的に swap file 作成して増量してみたり (約40GB)、さらにあれもこれもと、調子に乗りすぎてしまったらしい。困ったものである。

　というわけで、遅まきながら作業領域は内部ディスク (SSD 約500GB) ではなく、外部ディスク (HDD 約4TB) のほうに移動させようと考えた。

　WSL で外部ディスクを使う際、WSL 専用にするならともかく、Windows と共存させることを考えると、どうすれば良いか悩むところ。

- パーティションを切って住み分けるか
- Windows の NTFS のまま、DrvFs で（/mnt/f/ のように）マウントして使うか
- Windows の NTFS のまま、そこに仮想ハードディスク (VHD/VHDX) を作って、それを Linux 用にフォーマットした上で、マウントして使うか

　メリット・デメリットなどはやってみなければ実感できない。
　特に書き込み・読み込み速度は。

　ということで、実際にやって、比べてみることにした。

　以下でやった内容（仮想ハードディスク (VHD/VHDX) を作り、それを ext4 でフォーマットし、マウントし、そして DrvFS と読み書きの速度を比較した）の記録をメモとして残す。
　その際見つかった三つのハマりポイントについても、回避策・対応策とともに記載する。

　なお、パーティションを切ってしまうと元に戻すのは大変なので、今回はそれはパス。

# 表記について
　これ以下で実行内容を記述していくわけだが、その際に用いるコマンドの実行は Windows（Git Bash を利用）での実行と、WSL（Linux）での実行の二種類がある。

　極力勘違いを防ぐ目的でプロンプトを分ける。

- Windows での実行は `git-bash>` を使う。
- WSL（Linux）での実行は `wsl-linux>` を使う。

# 仮想ハードディスク作成

　なにはともあれ、まずは仮想ハードディスクを用意しなければ始まらない。
　というわけで、その作成であるが、仮想ハードディスクの作成からマウントまで、[こちら（リンク）][link-1] の記事を参考にさせていただいた。（大変助かりました。ありがとうございます！）

https://qiita.com/Sapphirus9/items/59075f1eba781c9b1330

　ほぼほぼ上記参考記事と同じような内容だが、一応私の実行した内容も以下に載せておく。

　特に、私が実行したところ、三つのハマりポイントが存在した。

- [ディスクの管理] で [VHDの作成] がグレイアウトする。
- コマンド wsl で unmount に失敗する。
- WSL を終了（shutdown など）するとマウントが解除されてしまう。

　これらの回避・対応にはそれなりに時間を要したので、もちろんその回避策・対応策も記載しておく。


## 仮想ハードディスク (VHD) の作成

　この作業は WSL ではなく Windows 側で行う。
　[スタート] を右クリックし、[ディスクの管理] を選択して起動する。

　さて、いきなり一つ目のハマりポイントだ。
　ここで私はしばらくハマってしまったのだが、メニューの [操作] をクリックして開けてみても、[VHDの作成]はグレイアウトしていて選択ができない。
　管理者権限がいるのか？　とかいろいろあれやこれやして時間を喰った。

　結局のところ、（私の場合）[ボリューム]の一番上が選択されているとグレイアウトする。逆に言うと、別のボリュームを選択したり、空いているところをクリックして一番上の選択を外せば、グレイアウトが解除され、無事に [VHDの作成] が選択できるようになった。（理由はよくわからない）

　[VHDの作成]が選択でき、ダイアログウィンドウが立ち上がれば、あとはそれほど迷うことは無いと思う。私の指定・選択を記述しておく。

 - 場所： `F:\WSL2\a001.vhdx` (フォーマットのほうを先に選択しておくとよい）
 - サイズ： 1TB
 - フォーマット： VHDX
 - 種類： 可変容量

　指定したら、[OK] をクリックして実行する。

　完了すると下のほうに「ディスクn」(nは数字) と新たなディスクが認識されているハズ。「初期化されていません」とか「未割り当て」などになっているだろう。そこを右クリックして「VHDの切断」をする。

「再接続するまで使用できなくなります」など言われるかもしれないが、[OK] で問題ない。むしろ切断しておかないと後でエラーが出てしまう。

## 作成した VHD を ext4 でフォーマット

　作成した VHD を、まずは WSL に接続する。
　ここの操作も Windows 側で行う。
　通常は PowerShell で実行するかもだが、私は Git Bash から実行した。

```shell-session:[Git Bash] VHD を WSL に接続させる
git-bash> wsl --mount --vhd f:\\wsl2\\a001.vhdx --bare
この操作を正しく終了しました。
```

　パスの指定のしかた（バックスラッシュの数とか）は PowerShell の場合と少し異なると思う。Git Bash からだと二つずつだが、PowerShell からだと一つだと思う。ご注意を。

　接続できたら、次は WSL 側（つまり Linux 側）から確認してみる。

```shell-session:Linux で接続確認
wsl-linux> lsblk
NAME MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda    8:0    0 363.3M  1 disk
sdb    8:16   0     2G  0 disk [SWAP]
sdc    8:32   0     1T  0 disk /snap
                               /mnt/wslg/distro
                               /
sdd    8:48   0     1T  0 disk
```

　私の場合、上記のように __sdd__ として認識されている。

　もしこの __sdd__ に対してパーティションを切るのであれば、`fdisk` を使ってパーティションを設定してから ext4 フォーマットする。

　私はパーティションを切らずにそのまま使う。

```shell-session:ext4でフォーマット
wsl-linux> sudo mkfs -t ext4 /dev/sdd
mke2fs 1.46.5 (30-Dec-2021)
Discarding device blocks: done
Creating filesystem with 268435200 4k blocks and 67108864 inodes
Filesystem UUID: a84caa6d-7cc8-48a0-90a2-e0865bba3c0f
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,         4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848

Allocating group tables: done
Writing inode tables: done
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done
```

　無事フォーマットできたら、ここでいったん WSL への接続を解除する。
　この操作は PowerShell や Git Bash など Windows 側から実行する。

```shell-session:[Git Bash] WSLから接続の解除……のハズなんだけど
git-bash> wsl --unmount f:\\wsl2\\a001.vhdx
指定されたファイルが見つかりません。
Error code: Wsl/Service/DetachDisk/ERROR_FILE_NOT_FOUND
```

　へっ？　わっと！？

　そう。ここでハマりポイント二つ目である。

　何故 `--mount` と同じパスを指定しているのに `--unmount` ではエラーになるんだ？

　少し悩んで、結局次のようにして（ここでは）回避した。

```shell-session:[Git Bash] WSL から接続の解除
git-bash> cd f:\\wsl2
git-bash> ls -l a001.vhdx
-rw-r--r-- 1 yynet2022 197121 17855152128 10月 15 15:00 a001.vhdx
git-bash> wsl --unmount .\\a001.vhdx
この操作を正しく終了しました。
```

　（この問題に関しては後述する）

## ext4 でフォーマットした VHD を WSL にマウント

　では改めて WSL にマウントする。
　この操作も Windows 側で実行する。
　その際、名前を指定する。どのような名前で Linux 側にマウントさせるのか、だ。

```shell-session:[Git Bash] WSL にマウント
git-bash> wsl --mount --vhd f:\\wsl2\\a001.vhdx --name work001
ディスクは '/mnt/wsl/work001' として正常にマウントされました。
注: /etc/wsl.conf で automount.root 設定を変更した場合、場所は異なります。
ディスクのマウントを解除してデタッチするには、'wsl.exe --unmount \\?\F:\WSL2\a001.vhdx' を実行してください。
```

　正常にマウントされたらしい。
　WSL 側 (つまり Linux 側）で確認してみる。

```shell-session: Linux でマウントを確認
wsl-linux> df /mnt/wsl/work001/
Filesystem      1K-blocks  Used  Available Use% Mounted on
/dev/sdd       1055761844    28 1002058392   1% /mnt/wsl/work001
```

　大丈夫そう。

## unmount でのエラーについて

　最後のマウントしたとき、`wsl` コマンドはヒントをくれていた。
　そう。以下の文だ。

```
...デタッチするには、'wsl.exe --unmount \\?\F:\WSL2\a001.vhdx' を実行してください。
```

　なるほど。
　ではそれに倣ってやってみるとしよう。

```shell-session:[Git Bash] unmount 再挑戦 その１
git-bash> wsl --unmount '\\?\f:\wsl2\a001.vhdx'
この操作を正しく終了しました。
```

　ここでは Git Bash から実行しているので `''` で囲っている。
　もし `''` を使わないとしたらこうなる。

```shell-session:[Git Bash] unmount 再挑戦 その２
git-bash> wsl --unmount \\\\?\\f:\\wsl2\\a001.vhdx
この操作を正しく終了しました。
```


# 実験・計測

　さて、ではメインディッシュといこう。
　書き込み、および読み込み速度の計測・比較である。

　改めて記すが、私の Windows 機には以下の二つのドライブがある。

- C ドライブ（内蔵 500GB SSD / NVMe M.2 (PCIe 3.0×4), NTFS）
- F ドライブ（外付 4TB HDD / USB 3.0, NTFS）

　今回の実験・計測では、それぞれのドライブで、

- __ext4/VHDX__: ext4 でフォーマットした VHDX をマウントした場所での読み書き
- __DrvFs__: Windows のドライブ（C: や F:）を DrvFS でマウントした場所（/mnt/c や /mnt/f）での読み書き
- __Git Bash__: 比較として WSL (Linux) じゃなく、Git Bash からそれぞれの場所での読み書き

　の三種ずつ、合計六種の計測を行った。

　どうやったかとか、ログなどは後述するとして、いきなりだが結果を示す。

|            | 書き込み速度 | 読み込み速度 |
|------------|--------------|--------------|
|C: ext4/VHDX|   2.1 GB/s   |   2.8 GB/s |
|C: DrvFs    | 179.0 MB/s   | 467.0 MB/s |
|C: Git Bash | 235.0 MB/s   | 966.0 MB/s |
|F: ext4/VHDX|   2.0 GB/s   |   2.6 GB/s |
|F: DrvFs    |  86.1 MB/s   | 138.0 MB/s |
|F: Git Bash | 190.0 MB/s   | 182.0 MB/s |

　ext4/VHDX の圧勝である。
　文字通り桁が違う。

　が、正直、「あれ？　おかしくね？」とは思った。

　Cドライブのほうは（前述したが PCI 3.0×4 なので）ともかく、Fドライブのほうは __USB 3.0__ である。
　USB 3.0 の最大転送速度は 5Gbps のハズ。つまり 640MB/s だ。
　だから「そんなスピード出るわけねぇ」と思ったわけだ。
　でも、何度計測してもほとんど変化なかった。

　つまり、VHDX か ext4 か、もしくは両方かもしれないし別の何かかもしれないが、よく分からないマジックを使ってくれているのだろう。ありがたいことだ。嬉しい誤算というヤツだ。（仕組みは全然分からんけど）

　というわけで、WSL から作業領域として使うなら、速度だけじゃなくパーミッションの扱い（Linux との相性）とか考えると、もはや ext4/VHDX 一択だと思う。
　DrvFS は、あくまで Windows の領域を参照できるモノとして（そういう意味ではとても便利なモノとして）、考えておくのが良いのではないだろうか。

　……なのだが、ここで一つ問題が浮上。
　追加した ext4/VHDX （私の場合 `f:\wsl2\a001.vhdx`）のマウントが維持できないのだ。

　WSL をログアウトして終了したり shutdown したりすると、マウントが解除されてしまうことに気付いた。どうやらそういう仕様らしい。

　「え？　毎回 `wsl --mount` しないといけないの？　マジで？」と、愕然としてしまった。

　これを調べ回ったのがハマりポイント三つ目。
　回避策・対応策は後述する。

## 実験・計測ログ

　計測方法はいたってシンプル。 20GB のファイル読み書きである。

　何故 20GB なのか？
　私の環境 WSL は 8GB のメモリと 2GB のスワップが設定されている。

```shell-session:freeコマンドによるメモリ量の確認
wsl-linux> free
               total        used        free      shared  buff/cache   available
Mem:         7926928      509800     1190524        3200     6226604     7113880
Swap:        2097152         268     2096884
```

　したがって、少なくともこれより大きいサイズ（できれば倍のサイズ）を扱うことでファイルキャッシュの影響を極力受けないようにしたいと考えた。

　読み書きにはコマンド `dd` を使う。

　では、それぞれのログを示していく。まずは C ドライブの書き込み。

```shell-session:C: ext4/VHDX 書き込み
wsl-linux> time dd if=/dev/zero of=/home/yynet2022/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 10.2215 s, 2.1 GB/s

real    0m10.313s
user    0m0.000s
sys     0m10.104s
```

```shell-session:C: DrvFS 書き込み
wsl-linux> time dd if=/dev/zero of=/mnt/c/Users/yynet2022/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 119.794 s, 179 MB/s

real    1m59.802s
user    0m0.011s
sys     0m6.759s
```

```shell-session:C: Git Bash 書き込み
git-bash> time dd if=/dev/zero of=$HOME/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 91.3678 s, 235 MB/s

real    1m31.490s
user    0m0.578s
sys     0m3.093s
```

　次に F ドライブの書き込み。

```shell-session:F: ext4/VHDX 書き込み
wsl-linux> time dd if=/dev/zero of=/mnt/wsl/work001/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 10.8702 s, 2.0 GB/s

real    0m11.211s
user    0m0.000s
sys     0m10.807s
```

```shell-session:F: DrvFS 書き込み
wsl-linux> time dd if=/dev/zero of=/mnt/f/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 249.36 s, 86.1 MB/s

real    4m9.368s
user    0m0.000s
sys     0m11.161s
```

```shell-session:F: Git Bash 書き込み
git-bash> time dd if=/dev/zero of=/f/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 113.184 s, 190 MB/s

real    1m53.422s
user    0m0.609s
sys     0m6.343s
```

次に読み込み。まずは C ドライブから。

```shell-session:C: ext4/VHDX 読み込み
wsl-linux> time dd of=/dev/null if=/home/yynet2022/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 7.70563 s, 2.8 GB/s

real    0m7.726s
user    0m0.000s
sys     0m4.447s
```

```shell-session:C: DrvFS 読み込み
wsl-linux> time dd of=/dev/null if=/mnt/c/Users/yynet2022/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 46.0273 s, 467 MB/s

real    0m46.037s
user    0m0.000s
sys     0m3.299s
```

```shell-session:C: Git Bash 読み込み
git-bash> time dd of=/dev/null if=$HOME/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 22.2325 s, 966 MB/s

real    0m22.355s
user    0m0.000s
sys     0m9.780s
```

　最後に F ドライブの読み込み。

```shell-session:F: ext4/VHDX 読み込み
wsl-linux> time dd of=/dev/null if=/mnt/wsl/work001/foo  bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 8.42033 s, 2.6 GB/s

real    0m8.441s
user    0m0.000s
sys     0m4.453s
```

```shell-session:F: DrvFS 読み込み
wsl-linux> time dd of=/dev/null if=/mnt/f/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 155.614 s, 138 MB/s

real    2m35.619s
user    0m0.000s
sys     0m6.706s
```

```shell-session:F: Git Bash 読み込み
git-bash> time dd of=/dev/null if=/f/foo bs=1024M count=20
20+0 records in
20+0 records out
21474836480 bytes (21 GB, 20 GiB) copied, 117.944 s, 182 MB/s

real    1m58.071s
user    0m0.000s
sys     0m9.686s
```

　以上である。

# 追加した ext4/VHDX の自動マウント設定

　いろいろ調べまわったが、shutdown などで WSL が停止すると、マウント解除することは避けられないらしい。

　マウントが維持できないのであれば、次善策として自動的にマウントしてほしい。毎回手動でマウントさせるのは面倒すぎる。勘弁してほしい。

　ひとつ、面白いことを知った。
　実は WSL から Windows コマンドを実行できるらしい。知らなかった。

　詳細は[こちら（リンク）][link-2]。「相互運用の設定」のところ。WSL での Windows プロセスの起動はデフォルトで有効になっているらしい。

　一応確認してみた。

```shell-session:WSL から wsl.exe を実行
wsl-linux> type wsl.exe
wsl.exe is /mnt/c/WINDOWS/system32/wsl.exe
wsl-linux> wsl.exe --list -v
  NAME            STATE           VERSION
* Ubuntu-22.04    Running         2
  Ubuntu          Stopped         2
```

　これができるならば話は早い。
　Linux側で、起動時にマウントするようにしておけばいい。
　例えば `/etc/init.d/` にシェルスクリプトを用意するなどして。

　でも、[同じページ][link-2]の解説で、 `/etc/wsl.conf` にブート時のコマンドを指定できることもわかったので、こちらを使うことにした。

```conf:/etc/wsl.conf

[boot]
systemd=true
command=/root/boot.sh
```

　`command=` で `/root/boot.sh` を実行するように指定。
　次に `/root/boot.sh` の内容を示す。

```shell:/root/boot.sh
#! /bin/sh
exec >/root/boot$$.log 2>&1

echo "$PATH" | grep -q /mnt/c/ || export PATH="$PATH:/mnt/c/Windows/System32"
echo "PATH: $PATH"
set -x
wsl.exe --list -v
wsl.exe --mount --vhd 'f:\wsl2\a001.vhdx' --name work001
```

　内容はこんな感じ。
　このファイルに実行パーミッションを付けることを忘れちゃいけない。

```shell-session:/root/boot.sh に実行パーミッションを付ける
wsl-linux> sudo chmod +x /root/boot.sh
wsl-linux> sudo ls -l /root/boot.sh
-rwxr-xr-x 1 root root 219 Oct 16 19:19 /root/boot.sh
```

　これで、WSL を終了させても、次に起動する際には自動的にマウントされる。

　ただし、ひとつ懸念がある。
　私はそういう使い方をしていないのだが、WSL で複数のディストリビューションを頻繁に使っている場合だ。

　WSL のヘルプによると `--mount` オプションは、

 >すべての WSL 2 ディストリビューションで物理ディスクまたは仮想ディスクを アタッチしてマウントします。

　という動作をするらしい。つまり、一度 `--mount` を実行すれば、全てのディストリビューションでアタッチ＆マウントされる。

　したがって、全てのディストリビューションで上記 `/etc/wsl.conf` などの設定をしてしまうと……どうなるんだろう？
　（ログにエラーが出るだけで実害が無いならばいいんだけれど）

# まとめ
　仮想ハードディスクの作成から Linux へのマウントまで実施し、そして速度比較を行ってみた。

　その結果、WSL で仮想ハードディスク (VHDX) の利用はかなりお勧めだと感じた。作業領域として考えた場合、少なくとも DrvFS に比べてあの読み書きの速度はかなり魅力的だと思う。

# 参考サイト
1. [WSL2で追加できるVHDを作って保存先にしたい][link-1]
2. [WSL での詳細設定の構成][link-2]

[link-1]: https://qiita.com/Sapphirus9/items/59075f1eba781c9b1330
[link-2]: https://learn.microsoft.com/ja-jp/windows/wsl/wsl-config