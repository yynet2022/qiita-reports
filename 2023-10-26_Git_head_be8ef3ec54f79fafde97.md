<!--
title:   [Git] ローカルブランチ専用リポジトリを作成 [備忘録]
tags:    Git,head,ブランチ,リポジトリ
id:      be8ef3ec54f79fafde97
private: false
-->
# はじめに

　Git を使っていて、メインとなるリポジトリとは別に、第二のリポジトリを作成・追加して使いたくなることがある。

　例えば、
 - GitHub で公開されている便利なオープンソースを使わせてもらい、だけど自分たちに合わせたカスタマイズをして、それを社内リポジトリで共有する、とか。
 - チームで開発しているプロジェクトのリポジトリとは別に、自分専用のリポジトリを作成・追加して、そこにバックアップも取りつつ、雑多な試行錯誤した上で、ある程度形になったらチームのリポジトリのほうに持っていく、とかとか。

　よくある話だと思う。（たぶん）
　そのような場合、メインのリポジトリでは `main` や `master` などのブランチ名は使われているだろうから、混乱しないように別のブランチ名（例えば `dev` とか `local` とか）を使って別リポジトリに push したいのだが、それには少しばかりコツがいるようなので、備忘録としてメモしておく。

　ざっと検索してみると、「自分の作業環境に複数のリモートリポジトリを追加する方法」などの記事は多く見かけるが、本記事はどちらかというと「（二つ目以降の）リモートリポジトリの作成方法」のほうに主眼を置いたものである。

# 要約（もしくは結論）

　本記事は備忘録が主目的なので、思い出せるように、そして再現しやすいように、少し細かくメモしている。

　要は、リモートリポジトリの HEAD ブランチ名（カレントブランチ名とかデフォルトブランチ名などとも言うらしい）をちゃんと設定しておく必要があり、その設定方法と、それをしないとどういうエラーが生じるかのメモである。

　結論を先に記すと、Git 2.27 以前、またはリモートリポジトリ作成後であれば `symbolic-ref HEAD` を使い（ケース１）、Git 2.28 以降では `init -b <branch-name>` が使えることを利用する（ケース２）が便利。


# この記事での定義

 - __メインリポジトリ__：`main` や `master` などのブランチがある元々のリポジトリを指す。前章で言うところの、GitHub のリポジトリやチームで使っているリポジトリのこと。
 - __セカンドリポジトリ__：メインリポジトリとは別の、前章で言うところの、カスタマイズしたモノを格納している社内リポジトリや、個別に試行錯誤中のモノを格納する個人用リポジトリのこと。


# 想定する環境の準備と課題の確認

　メインリポジトリの例として、ファイルが一つだけ含まれている `/tmp/main.git` というリポジトリを用意して、それを使うこととする。

```shell-session:メインリポジトリ（サンプル）の作成
$ git init --bare /tmp/main.git
Initialized empty Git repository in /tmp/main.git/
$ git clone /tmp/main.git/
Cloning into 'main'...
warning: You appear to have cloned an empty repository.
done.
$ cd main
$ touch a.txt
$ git add .
$ git commit -m 'First commit.'
[main (root-commit) 3a491d9] First commit.
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 a.txt
$ git push
Enumerating objects: 3, done.
Counting objects: 100% (3/3), done.
Writing objects: 100% (3/3), 212 bytes | 212.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
To /tmp/main.git/
 * [new branch]      main -> main
```

　さて、ではこれを使って作業を行っていくとしよう。
　まずは作業領域に移動し、`/tmp/main.git` を `clone` する。

```shell-session:作業領域で main.git を clone
$ cd /work/
$ git clone /tmp/main.git
Cloning into 'main'...
done.
$ cd main/
$ ls
a.txt
```

　含まれているのは（当然ながら） `a.txt` だけ。
　ブランチやリモートリポジトリも確認しておく。

```shell-session:ブランチやリモートリポジトリを確認
$ git branch -a -v    # ブランチの確認
* main                3a491d9 First commit.
  remotes/origin/HEAD -> origin/main
  remotes/origin/main 3a491d9 First commit.
$ git remote -v       # リモートリポジトリの確認
origin  /tmp/main.git (fetch)
origin  /tmp/main.git (push)
```

　ブランチは `main` だけ、リモートリポジトリも `/tmp/main.git` だけであることが確認できる。そこに、個人的な開発用ブランチ `dev` を作成して、簡単な変更をコミットする。

```shell-session:ブランチ dev を作成して更新
$ git checkout -b dev           # ブランチ dev を作成して dev に移行
Switched to a new branch 'dev'
$ git branch -a -v              # ブランチと最新ログの確認
* dev                 3a491d9 First commit.
  main                3a491d9 First commit.
  remotes/origin/HEAD -> origin/main
  remotes/origin/main 3a491d9 First commit.
$ touch b.txt                   # ファイル b.txt を作成
$ git add .                     # それを add して、
$ git commit -m 'Add b.txt'     # コミットする (ブランチ dev へ）
[dev 8d1c528] Add b.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 b.txt
$ git branch -a -v              # ブランチと最新ログの確認
* dev                 8d1c528 Add b.txt
  main                3a491d9 First commit.
  remotes/origin/HEAD -> origin/main
  remotes/origin/main 3a491d9 First commit.
$ git log --oneline             # 最新ログの確認
8d1c528 (HEAD -> dev) Add b.txt
3a491d9 (origin/main, origin/HEAD, main) First commit.
```

　さて、この `dev` というブランチを、バックアップも取りたいし、一部の人たちと共有したい場合もある。なので自分の手元だけじゃなく、リモートリポジトリに push したい。しかし多くの人がアクセスするメインリポジトリにはしたくない。

　というわけで、セカンドリポジトリを作ってそっちにコミットしておきたい、というのが今回の話である。


# ケース１

　Git 2.27 以前を使っているか、またはリモートリポジトリをすでに作成した後であればこちらのケースになる。

　セカンドリポジトリとして、ここでは `my.git` という名前でリポジトリを作成してみる。

```shell-session:セカンドリポジトリの作成
$ cd /tmp/
$ git init --bare my.git
Initialized empty Git repository in /tmp/my.git/
```

　この `my.git` を、二つ目のリモートリポジトリとして作業領域に設定し、 `dev` をコミットする。


```shell-session:セカンドリポジトリへ push
$ cd /work/main/
$ ls
a.txt  b.txt
$ git remote add myrepo /tmp/my.git/  # myrepo という名前を付けて登録
$ git remote -v                       # 登録できたか確認
myrepo  /tmp/my.git/ (fetch)
myrepo  /tmp/my.git/ (push)
origin  /tmp/main.git (fetch)
origin  /tmp/main.git (push)
$ git branch -a -v                    # ブランチの内容確認
* dev                 8d1c528 Add b.txt
  main                3a491d9 First commit.
  remotes/origin/HEAD -> origin/main
  remotes/origin/main 3a491d9 First commit.
$ git push -u myrepo dev              # dev を myrepo に push する
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 12 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (5/5), 415 bytes | 415.00 KiB/s, done.
Total 5 (delta 0), reused 0 (delta 0), pack-reused 0
To /tmp/my.git/
 * [new branch]      dev -> dev
Branch 'dev' set up to track remote branch 'dev' from 'myrepo'.
$ git branch -a -v
* dev                 8d1c528 Add b.txt
  main                3a491d9 First commit.
  remotes/myrepo/dev  8d1c528 Add b.txt
  remotes/origin/HEAD -> origin/main
  remotes/origin/main 3a491d9 First commit.
```

　うまく `my.git` にブランチ `dev` を push できた、……ように見える。

　実際、__ほぼ__ うまくできている。

```shell-session:リモートリポジトリの状態を確認
$ git remote show myrepo
* remote myrepo
  Fetch URL: /tmp/my.git/
  Push  URL: /tmp/my.git/
  HEAD branch: (unknown)                # 注目すべき箇所
  Remote branch:
    dev tracked
  Local branch configured for 'git pull':
    dev merges with remote dev
  Local ref configured for 'git push':
    dev pushes to dev (up to date)
```

　上記で `'HEAD branch: (unknown)'` と出てしまっていること以外は。
　この状態でいつものように clone すると、

```shell-session:セカンドリポジトリを clone してみる
$ git clone /tmp/my.git/
Cloning into 'my'...
done.
warning: remote HEAD refers to nonexistent ref, unable to checkout.

$ ls -a my/
.  ..  .git
```

　このように warning が出て、ディレクトリの中は空だ。
　もちろんこれがどういう状況なのか分かっていれば、その後、以下のように手動でブランチ `dev` をチェックアウトして、問題なく使うこともできる。

```shell-session:ブランチ dev を明示的にチェックアウトして使う
$ cd my/
$ git checkout dev
Branch 'dev' set up to track remote branch 'dev' from 'origin'.
Switched to a new branch 'dev'
$ ls
a.txt  b.txt
$ git branch -a -v
* dev                8d1c528 Add b.txt
  remotes/origin/dev 8d1c528 Add b.txt
```

　もしくは clone する際に、引数 `-b <branch-name>` を付けてもよい。

```shell-session:clone に引数 -b を使う例
$ git clone -b dev /tmp/my.git/
Cloning into 'my'...
done.
$ cd my/
$ ls
a.txt  b.txt
$ git branch -a -v
* dev                8d1c528 Add b.txt
  remotes/origin/dev 8d1c528 Add b.txt
```

　どちらにしても、普段に比べてひと手間増えるので、あまりよろしい状態とは言えない。（と思う）

　この原因はこれである。

```shell-session:セカンドリポジトリの HEAD を確認
$ cd /tmp/my.git/
$ cat HEAD
ref: refs/heads/main
```

　リポジトリを作る際、HEAD ブランチにデフォルトのブランチ名（ここでは main）が使われているため、`dev` という名前のブランチしかない状態ではうまく動作しないのである。

　これをうまく動作するように修正するには、`symbolic-ref` を使って `HEAD` を修正する必要がある。（ファイル HEAD を直に修正することはお勧めしない。それでもできるけど）

```shell-session:セカンドリポジトリの HEAD を修正
$ cd /tmp/my.git/
$ cat HEAD
ref: refs/heads/main
$ git symbolic-ref HEAD refs/heads/dev
$ cat HEAD
ref: refs/heads/dev
```

　これで大丈夫。
　もう一度 `my.git` を clone してみる。


```shell-session:確認
$ git clone /tmp/my.git/
Cloning into 'my'...
done.
$ cd my
$ ls
a.txt  b.txt
$ git branch -a -v
* dev                 8d1c528 Add b.txt
  remotes/origin/HEAD -> origin/dev
  remotes/origin/dev  8d1c528 Add b.txt
$ git remote show origin
* remote origin
  Fetch URL: /tmp/my.git/
  Push  URL: /tmp/my.git/
  HEAD branch: dev                 # ちゃんと dev になってる
  Remote branch:
    dev tracked
  Local branch configured for 'git pull':
    dev merges with remote dev
  Local ref configured for 'git push':
    dev pushes to dev (up to date)
```

　上記で `'HEAD branch: dev'` となっていることも確認できる。


# ケース２

　Git 2.28 以降を使っていて、これからリポジトリを作成する場合は、こちらのケースで対応できる。

　Git 2.28 以降は、`git init` に、オプション `-b <branch-name>` が追加された。

　つまり `my.git` を作成する際に、HEAD ブランチ名を指定できる。

```shell-session:init で引数 -b を使う例
$ git init --bare -b dev my.git
Initialized empty Git repository in /tmp/my.git/
```

　これでうまく動作する。

　あとの操作はケース１と違いは無いので割愛する。
　その代わりに、`Git init` の引数 `-b` の説明を引用しておく。

>-b \<branch-name>
>--initial-branch=\<branch-name>
>Use the specified name for the initial branch in the newly created repository. If not specified, fall back to the default name (currently master, but this is subject to change in the future; the name can be customized via the init.defaultBranch configuration variable).


　ざっくりと訳してみた。

>>-b \<ブランチ名>
>>--initial-branch=\<ブランチ名>
>>新しく作成されたリポジトリでの最初のブランチに、指定された名前を使用する。指定しない場合は、デフォルト名を使う (現在は `master` だが、これは将来変更される可能性がある。名前は config 変数 `init.defaultBranch` を使用してカスタマイズできる)。


# 終わりに

　セカンドリポジトリを使う際の、ちょっとしたコツを示した。

　たぶん、セカンドリポジトリに GitHub や GitLab を使用しているならば、サーバのほうがうまく処理してくれて、このような問題は生じないかもしれない。

　セカンドリポジトリとして、自分で bare リポジトリを作成する際には、参考になれば幸いである。


# 参考サイト
- [Git init](https://git-scm.com/docs/git-init)
- [Git symbolic-ref](https://git-scm.com/docs/git-symbolic-ref)
- [Git - Git の参照](https://git-scm.com/book/ja/v2/Git%E3%81%AE%E5%86%85%E5%81%B4-Git%E3%81%AE%E5%8F%82%E7%85%A7)