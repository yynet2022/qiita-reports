<!--
title:   find と sed で tree 表示
tags:    Linux,ShellScript,find,sed,tree
id:      c758cb295629ddf6f080
private: false
-->
# 例

```shell-session
$ find . | sed -e 's,[^/]*/,|  ,g;s,^\(\(|  \)*\)|  ,\1|--,'
.
|--CMakeCache.txt
|--CMakeFiles
|  |--3.22.1
|  |  |--CMakeCXXCompiler.cmake
|  |  |--CMakeDetermineCompilerABI_CXX.bin
|  |  |--CMakeSystem.cmake
|  |  |--CompilerIdCXX
|  |  |  |--a.out
|  |  |  |--CMakeCXXCompilerId.cpp
|  |  |  |--tmp
|  |--a.out.dir
|  |  |--build.make
|  |  |--cmake_clean.cmake
|  |  |--compiler_depend.make
|  |  |--compiler_depend.ts
|  |  |--depend.make
|  |  |--DependInfo.cmake
|  |  |--flags.make
|  |  |--link.txt
|  |  |--progress.make
|  |--cmake.check_cache
|  |--CMakeDirectoryInformation.cmake
|  |--CMakeOutput.log
|  |--CMakeTmp
|  |--hello.dir
|  |  |--build.make
|  |  |--cmake_clean.cmake
|  |  |--compiler_depend.make
|  |  |--compiler_depend.ts
|  |  |--depend.make
|  |  |--DependInfo.cmake
|  |  |--flags.make
|  |  |--link.txt
|  |  |--progress.make
|  |--Makefile.cmake
|  |--Makefile2
|  |--progress.marks
|  |--TargetDirectories.txt
|--cmake_install.cmake
|--Makefile
```

# 解説

　find は、指定ディレクトリの階層をたどってファイル等を探すコマンド。

　検索条件を指定するコマンド引数はたくさんあるが、ここでは全てのファイルやディレクトリを表示させたいので、検索開始するディレクトリだけ指定している。

　昔、(Linux じゃない）別の UNIX 系 OS の標準 find では、コマンド引数 `-print` を指定しないと表示してくれないモノもあった（ハズ。記憶が不確かだけど）が、現在ほとんどの find は、デフォルトで表示してくれる（たぶん）。

　さらに言えば、検索開始ディレクトリの指定も、デフォルトでカレントディレクトリが使用される。したがって指定する必要無いのかもしれないが、でもそこは明示しておいたほうが、見る側としては分かりやすいので、ちゃんと指定することをお勧めする。

　sed は、ストリームエディタと呼ばれるコマンド。
　ここでは find の出力を読み込み、編集して出力するために使われている。

　エディタだけあって、編集用コマンドは多彩にある。
　ここでは s コマンドだけ使用している。

　s コマンドは以下のような書式を目にすることが多いかもしれない。

s/regexp/replacement/

　だが、必ずしも `/` である必要はない。
　特に今回のようなディレクトリパスを編集する場合は、ディレクトリの区切りの `/` とバッティングするため、別の文字を使ったほうが良いので `,` を使っている。

　使い方は `regexp` に一致する文字列を `replacement` に置き換えるというシンプルなもの。
　だが `regexp` のところは正規表現が使えるので、かなり奥の深いコマンドである。

`s,[^/]*/,|  ,g`

　「"/"でない 0 個以上の文字列と、それに続く"/"」という文字列を、「`|  `」という文字列に置き換える。

`s,^\(\(|  \)*\)|  ,\1|--,`

　「"`|  `"という 0 個以上の文字列(1)と、それに続く "`|  `"」という文字列を、(1)と「`|--`」という文字列に置き換える。

　この二つを `;` で連結させている。

# 終わりに

　正直、「車輪の再発明」だな、とは思ってる。

　でも、やっぱり自分で考えるの好きだし、考えたらメモ残しておきたいのがエンジニアという生き物の性分だし。（震え声

　というわけで、上記を少しいじって以下のような形で `~/.bashrc` にでも入れておけば、少しだけ幸せになれる……かもしれない？

```shell
lstree()
{
    test "x${1}" = x && set "."
    echo "$1"
    (cd "$1" && find . | sed -e '1d;s,[^/]*/,|  ,g;s,^\(\(|  \)*\)|  ,\1|--,')
}
```