# パッケージとクレート

> 参考: https://doc.rust-lang.org/stable/book/ch07-01-packages-and-crates.html

このセクションで最初にパッケージとクレートを取り上げます。

クレートは、Rustコンパイラが一度に考慮する最小のコード量です。`cargo`ではなく、`rustc`を実行し単一のソースコードを渡した場合でも、コンパイラはそのファイルをクレートとみなします。
クレートはモジュールを含むことができ、モジュールはクレートと一緒にコンパイルされ、他のファイルで定義されることがあります。

クレートには、バイナリクレートとライブラリクレートいう2つの形態があります。バイナリクレートは、コマンドラインプログラムやサーバーなど、実行可能なファイルにコンパイルすることができるプログラムです。
それぞれ、実行ファイルが実行されたときに何が起こるかを定義する`main`関数を持たなければなりません。これまで作成したクレートは全てバイナリクレートです。

ライブラリのクレートは、メイン関数を持たず、実行ファイルにコンパイルされません。その代わり複数のプロジェクトで共有することを目的とした機能を定義しています。
例えば第2章の`rand crate`は、他のRustceansがクレートという場合、ほとんどがライブラリのクレートという意味で、一般的なプログラミングの概念であるライブラリと互換性を持ってクレートを使っています。

`crate`ルートはRustコンパイラが起動するソースファイルで、`crate`のルートモジュールを構成します。

パッケージは、一連の機能を提供する1つまたは複数のクレートのバンドルです。パッケージには、それらのクレートをビルドする方法を記述した`Cargo.toml`ファイルが含まれます。
Cargoは実際にはコードをビルドするために使ってきたコマンドラインツールのバイナリクレートを含むパッケージです。Cargoパッケージは、バイナリクレートが依存するライブラリクレートも含んでいます。
他のプロジェクトは、Cargoコマンドラインツールが使用するのと同じロジックを使用するために、Cargoライブラリのクレートに依存することができます。

パッケージは好きなだけバイナリクレートを含むことができますが、ライブラリ、バイナリクレートであれ、少なくとも1つのクレートを含めなくてはなりません。

プロジェクトディレクトリの中には、Cargo.tomlというファイルがあり、パッケージが作成されています。またsrcディレクトリには、main.rsが含まれています。
Cargo.tomlを開いて、src/main.rsの記述がないことを確認してください。Cargoは、src/main.rsはパッケージと同じ名前のバイナリクレートのクレートルートであるという慣例に従っています。
同様にCargoは、パッケージディレクトリにsrc/lib.rsがあれば、そのパッケージにはパッケージと同じ名前のライブラリクレートがあり、src/lib.rsはそのクレートルートであることを認識します。
Cargoは、ライブラリやクレートを構築するために、`rustc`にクレートルートファイルを渡します。

ここでは、src/main.rsだけを含むパッケージがあり、それは`project-name`という名前のバイナリクレートだけを含むことを意味します。
パッケージがsrc/main.rs, src/lib.rsを含んでいる場合、パッケージと同じ名前のバイナリとライブラリの2つのクレートを持っています。
パッケージは、src/binディレクトリにファイルを配置することで、複数のバイナリクレートを持つことができます。各ファイルは、別々のバイナリクレートになります。