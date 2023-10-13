<!--
title:   Hello World から始める CMake (3)
tags:    C++,CMake,SOVERSION,shared,static
id:      cd0f27b13269dd96f6f9
private: false
-->
# はじめに
　前回は "Hello, world!" を出力する C++ プログラムを、cmake で構築するための最小限の設定と実行方法を紹介した。

https://qiita.com/yynet2022/items/be92158d8c69e24a71e0

　今回はそれをライブラリ化してみる。
　その際に必要と思われる、以下について紹介する。

- 静的ライブラリと共有ライブラリ、 __両方一緒に__ 構築する方法。
- バージョンを付けた共有ライブラリ (例えば _*.so.1_ とか _*.so.1.0.3_ とか）の構築方法。
- インストールについての指定方法。

# 例
　ここでは余計な情報を省いて読みやすく、必要な情報を得やすくするため、ヘッダファイルは使わずに簡便なコードにしている。なので「こんなコードの書き方はなってねぇっ！」とか言う指摘はご容赦を。

```c++:main.cxx
extern int hello();
int main(int argc, char *argv[])
{
  hello();
  return 0;
}
```

```c++:hello.cxx
#include <iostream>
int hello()
{
  std::cout << "Hello, world!" << std::endl;
  return 0;
}
```
　お察しの通り、関数 `hello()` が含まれる _hello.cxx_ でライブラリを作り、_main.cxx_ とリンクして実行ファイルを作成する。
　その構築をするための _CMakeLists.txt_ が以下になる。

```cmake:CMakeLists.txt
# 下限バージョンを 3.16 に指定する
cmake_minimum_required(VERSION 3.16)

# プロジェクト名、バージョン、コンパイル言語などを指定
project(HelloWorld VERSION 0.3.1.21 LANGUAGES CXX)

# プロジェクト名とバージョンを表示する
message("This project is ${PROJECT_NAME} version ${PROJECT_VERSION}")

# 共有ライブラリの設定
#   ライブラリは hello.cxx から作成されることを指定
add_library(hello_shared SHARED hello.cxx)
#   各プロパティの指定
set_target_properties(
  hello_shared PROPERTIES
  OUTPUT_NAME hello
  VERSION ${PROJECT_VERSION}
  SOVERSION ${PROJECT_VERSION_MAJOR})

# 静的ライブラリの設定
#   ライブラリは hello.cxx から作成されることを指定
add_library(hello_static STATIC hello.cxx)
#   各プロパティの指定
set_target_properties(hello_static PROPERTIES OUTPUT_NAME hello)

# 実行ファイル a.out は main.cxx から作成されることを指定
add_executable(a.out main.cxx)

# 実行ファイル a.out は hello_shared(共有ライブラリ) をリンクする
target_link_libraries(a.out hello_shared)

# インストール先を指定
install(TARGETS a.out RUNTIME DESTINATION bin)
install(TARGETS hello_shared LIBRARY DESTINATION lib)
install(TARGETS hello_static ARCHIVE DESTINATION lib)
```

# 実行
　解説は後にして、まずは実行してみる。
　今回はインストールまで行うのでコマンド引数 `--install-prefix` でインストールするディレクトリを指定する。ただしあくまでテストなので、カレントディレクトリに作成するよう指定している。

```shell-session:実行その１ ビルドシステム生成
$ ls
CMakeLists.txt  hello.cxx  main.cxx
$ cmake -S . -B build --install-prefix $(pwd)/top
-- The CXX compiler identification is GNU 11.4.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
This project is HelloWorld version 0.3.1.21
-- Configuring done
-- Generating done
-- Build files have been written to: /home/yynet2022/ctest/build
$ ls
CMakeLists.txt  build  hello.cxx  main.cxx
```
<details><summary>（補足）$(pwd)/top という指定について</summary>

ここでの `$(pwd)` は、バッククォートを使った `` `pwd` `` と同じ。
コマンド pwd を実行して、その出力（文字）を使っている。
つまりカレントディレクトリを絶対パスで指定している。

```shell-session:例
$ pwd
/tmp/foo
$ echo `pwd`/top
/tmp/foo/top
$ echo $(pwd)/top
/tmp/foo/top
```
　Bash ユーザの自分は、バッククォートよりこっち（`$(pwd)`）のほうが好み。
（補足　終わり）
</details>

　ちゃんとディレクトリ `build/` が作成されていることが分かる。
　次にプログラムの構築を実行する。

```shell-session:実行その２ プログラム構築
$ cmake --build build/
[ 16%] Building CXX object CMakeFiles/hello_shared.dir/hello.cxx.o
[ 33%] Linking CXX shared library libhello.so
[ 33%] Built target hello_shared
[ 50%] Building CXX object CMakeFiles/hello_static.dir/hello.cxx.o
[ 66%] Linking CXX static library libhello.a
[ 66%] Built target hello_static
[ 83%] Building CXX object CMakeFiles/a.out.dir/main.cxx.o
[100%] Linking CXX executable a.out
[100%] Built target a.out
$ ls build/
CMakeCache.txt  a.out                libhello.so
CMakeFiles      cmake_install.cmake  libhello.so.0
Makefile        libhello.a           libhello.so.0.3.1.21
$ ./build/a.out
Hello, world!
```

　問題なくプログラム `a.out` やライブラリ `libhello.*` が構築でき、プログラム `a.out` も動くことが確認できる。
　最後にインストールである。

```shell-session:実行３ プログラムのインストール
$ cmake --install build/
-- Install configuration: ""
-- Installing: /home/yynet2022/ctest/top/bin/a.out
-- Set runtime path of "/home/yynet2022/ctest/top/bin/a.out" to ""
-- Installing: /home/yynet2022/ctest/top/lib/libhello.so.0.3.1.21
-- Installing: /home/yynet2022/ctest/top/lib/libhello.so.0
-- Installing: /home/yynet2022/ctest/top/lib/libhello.so
-- Installing: /home/yynet2022/ctest/top/lib/libhello.a
$ lstree top/
top/
|--bin
|  |--a.out
|--lib
|  |--libhello.a
|  |--libhello.so
|  |--libhello.so.0
|  |--libhello.so.0.3.1.21
```

　ディレクトリ `top` 以下にインストールされたことが確認できる。

<details><summary>lstreeについて</summary>

コマンド `tree` と同じようにディレクトリ構成をツリー表示するもの。
「何このコマンド」と興味のある人は [こちら](2023-10-04_Linux_ShellScript_find_sed_tree_c758cb295629ddf6f080.md) をどうぞ。（宣伝ｗ）
</details>

# 解説
　_CMakeLists.txt_ の内容だが、見ていただければだいたい分かるかもしれないが、一応簡単に解説を付けてみる。

>`# 下限バージョンを 3.16 に指定する`
`cmake_minimum_required(VERSION 3.16)`

　コメントにある通り、cmake の下限バージョンを指定している。
　この指定をすると、cmake のバージョンが 3.16 より前だとエラーになって実行できない。

　逆に言うと「バージョン 3.16 で実装されている機能を使うぞ」と宣言していることになる。

　特に理由が無い限り、新しめのバージョンを指定しておくことをお勧めする。そうすればいろいろ便利な機能が使える。

:::note info
自分の場合、Qt6 を使うことが多いので 3.16 をよく指定している。ちなみに使用している cmake は 3.22.1 である。
:::

>`# プロジェクト名、バージョン、コンパイル言語などを指定`
`project(HelloWorld VERSION 0.3.1.21 LANGUAGES CXX)`

　これもコメントの通りプロジェクト名、バージョン、コンパイル言語を指定している。

:::note info
バージョン (VERSION) を指定するためには、`cmake_minimum_required()` で 3.0 以上を指定する必要がある。
:::

　ここで指定したバージョンを元に、 `PROJECT_VERSION` や `PROJECT_VERSION_MAJOR` などが自動的に設定される。
　ちなみに VERSION の指定は以下のような形式になる。

`VERSION <major>[.<minor>[.<patch>[.<tweak>]]]`

　それぞれを 0 以上の整数で指定する。負の値やアルファベットなどは不可。

>`# 共有ライブラリの設定`
`# 　ライブラリは hello.cxx から作成されることを指定`
`add_library(hello_shared SHARED hello.cxx)`
`# 　各プロパティの指定`
`set_target_properties(`
&emsp;&emsp;`hello_shared PROPERTIES`
&emsp;&emsp;`OUTPUT_NAME hello`
&emsp;&emsp;`VERSION ${PROJECT_VERSION}`
&emsp;&emsp;`SOVERSION ${PROJECT_VERSION_MAJOR})`

　`add_library()` で、_hello_shared_ という名前、共有ライブラリの指定、そしてソースファイルの指定をしている。

　次の `set_target_properties()` でプロパティの設定をしている。

　`OUTPUT_NAME` は出力するファイル名の指定。これにより `libhello_shared.so` ではなく `libhello.so` になる。

　`VERSION` で `*.so.${PROJECT_VERSION}` も作成するよう指定し、さらに `SOVERSION` で `*.so.${PROJECT_VERSION_MAJOR}` も作成するのと同時に、リンクする soname も指定している。

```shell-session:sonameの確認
$ ldd build/a.out | grep hello
        libhello.so.0 => /home/yynet2022/ctest/build/libhello.so.0 (0x00007fc365c72000)
```

　リンクが `libhello.so` ではなく `libhello.so.0` になっていることが確認できる。

　広く配布するような共有ライブラリの場合、共有ライブラリの API バージョン管理には、これが重要な役目を果たすことになる。（この辺、機会があれば別途紹介したいと思う）

>`# 静的ライブラリの設定`
`# 　ライブラリは hello.cxx から作成されることを指定`
`add_library(hello_static STATIC hello.cxx)`
`# 　各プロパティの指定`
`set_target_properties(hello_static PROPERTIES OUTPUT_NAME hello)`

　こちらは静的ライブラリの指定である。
　`add_library()` で指定する名前を _hello_static_ としていることで共有ライブラリとはバッティングしないようにしている。

>`# インストール先を指定`
`install(TARGETS a.out RUNTIME DESTINATION bin)`
`install(TARGETS hello_shared LIBRARY DESTINATION lib)`
`install(TARGETS hello_static ARCHIVE DESTINATION lib)`

　最後にインストール先を指定している。
　コマンド引数 `--install-prefix` で指定したディレクトリ以下の `bin` や `lib` にインストールすることを指示している。

# 終わりに

　駆け足での紹介になってしまったかもしれない。
　それでも長くなってしまったと思う。
　ので、さらなる詳細は以下の参考リンクを参照頂きたい。

# 参考リンク：
1. [CMake](https://cmake.org/)
2. [CMake Documentation and Community](https://cmake.org/documentation/)
3. [cmake(1) &mdash; CMake Documentation][link-3]
4. [project &mdash; CMake Documentation][link-4]
5. [add_executable &mdash; CMake Documentation][link-5]
6. [cmake_minimum_required &mdash; CMake Documentation][link-6]
7. [message &mdash; CMake Documentation][link-7]
8. [add_library &mdash; CMake Documentation][link-8]
9. [set_target_properties &mdash; CMake Documentation][link-9]
10. [target_link_libraries &mdash; CMake Documentation][link-10]
11. [install &mdash; CMake Documentation][link-11]
12. [cmake-properties(7) &mdash; CMake Documentation][link-12]

[link-3]: https://cmake.org/cmake/help/latest/manual/cmake.1.html
[link-4]: https://cmake.org/cmake/help/latest/command/project.html
[link-5]: https://cmake.org/cmake/help/latest/command/add_executable.html
[link-6]: https://cmake.org/cmake/help/latest/command/cmake_minimum_required.html
[link-7]: https://cmake.org/cmake/help/latest/command/message.html
[link-8]: https://cmake.org/cmake/help/latest/command/add_library.html
[link-9]: https://cmake.org/cmake/help/latest/command/set_target_properties.html
[link-10]: https://cmake.org/cmake/help/latest/command/target_link_libraries.html
[link-11]: https://cmake.org/cmake/help/latest/command/install.html
[link-12]: https://cmake.org/cmake/help/latest/manual/cmake-properties.7.html