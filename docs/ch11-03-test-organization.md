# テストの体系化

> Ref: https://doc.rust-jp.rs/book-ja/ch11-03-test-organization.html

テストは複雑な鍛錬であり、人によっては専門用語や体系化が異なります。
Rustのコミュニティでは、テストを2つの大きなカテゴリで捉えています。単体テストと結合テストです。
単体テストは、小規模でより集中してして、個別に1回の1モジュールをテストし、非公開のインターフェイスもテストすることがあります。
結合テストは、完全にライブラリ外になり、他の外部コード同様に自分のコードを使用し、公開インターフェイスのみ使用し、1テストにつき複数のモジュールを用いることもあります。

どちらのテストを書くのも、ライブラリの一部が個別かつ共同でして行うことを確認するのに重要なことです。

## 単体テスト

単体テストの目的は、各単位のコードをテストし、コードが想定通り動いたり動かなかったりする箇所を迅速に特定することです。
このテストは対象となるコードと共に各ファイルに置かれます。
慣習は`tests`モジュールを作成し、テスト関数を含み、`cfg(test)`と注釈することです。


**#[cfg(test)]** 

testsモジュールの`#[cfg(test)]`という注釈は、コンパイラに`cargo test`を走らせた時だけテストコードをコンパイルし走らせるように指示します。
これにより`cargo build`でライブラリをビルド時にはコンパイルタイムを節約し、コンパイル後のサイズも節約することができます。

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

`cfg`という属性は、**configuration**を表していて、ある特定の設定オプションが与えられたら含むようにコンパイラに指示します。
今回の場合、設定オプションは`test`で、`cfg`属性を使用することで、`cargo test`でテストを実行した場合のみ、Cargoがテストコードをコンパイルしています。

**非公開関数** 

テストコミュニティ内では、非公開関数を直接テストすべきか議論は、他の言語でのテストが困難だったり、不可能だったりします。
Rustの公開性規則により、非公開関数をテストすることは可能です。

```rust
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}

fn internal_adder(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn internal() {
        assert_eq!(4, internal_adder(2, 2));
    }
}
```

## 結合テスト

Rustにおいて、結合テストは完全にライブラリ外のものです。
他のコードと全く同様にライブラリを使用するので、ライブラリの公開APIの一部である関数しか呼び出すことはできません。
統合テストの目的は、ライブラリの色々な部分が共同で正常に動作しているかみることです。
単体では正常に動くコードも、結合した状態だと問題を孕む可能性もあるので、結合したコードのテストの範囲も同様に重要になるのです。
結合テストの作成は、`tests`ディレクトリが必要になります。

**testsディレクトリ** 

`tests`ディレクトリは、プロジェクトのトップ階層に作成します。
Cargoは、このディレクトリに結合テストのファイルを探すことを把握しています。
そして、このディレクトリ内にいくらでもテストファイルを作成することができ、Cargoはそれぞれのファイルを個別のクレートとしてコンパイルします。

`tests/integration_test.rs`ファイルを作成し、以下のコードを記述します。

```rust
extern crate testing;

#[test]
fn it_adds_two() {
    assert_eq!(4, testing::add_two(2));
}
```

`extern crate testing;`は、ライブラリをインポートする時に必要になります。

`tests/integration_test.rs`では`#[cfg(test)]`注釈は必要ありません。
Cargoは`tests`ディレクトリを特別に扱い、`cargo test`を走らせた時のみこのディレクトリのファイルをコンパイルします。

上記のコードの実行結果は次のとおりです。

```bash
cargo test

running 10 tests
test tests::it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_test.rs (target/debug/deps/integration_test-770fb491099cb155)

running 1 test
test it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests testing

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

単体テスト、結合テスト、ドックテストの順番で3つの出力があります。

また、`cargo test --test`の後にテストを行うファイル名を指定することで、そのファイルに書かれているテストのみを実行することもできます。

**サブモジュール**

結合テストを追加していくにつれ、`tests`ディレクトリに2つ以上のファイルを作成して体系化したくなるかもしれません。

各結合テストファイルをそれ自身のクレートとして扱うと、エンドユーザが自身のクレートを使用するかのように個別のスコープを生成するのに役立ちます。
これは`tests`ディレクトリのファイルが、コードをモジュールとファイルに分けるように、`src`のファイルとは同じ振る舞いを共有しないことを意味します。

`tests`ディレクトリのファイルの異なる振る舞いの1つに、`tests/common.rs`に`setup`という関数を配置するものがあります。
これは複数のテストファイルの複数のテスト関数から呼び出したい`setup`に何らかのコードを追加することができます。

```rust
pub fn setup() {
    // setup code specific to your library's tests would go here
}
```

`common`がテスト結果に表示されるのを防ぐには、`tests/common/mod.rs`を作成します。
ここでは`common`にサブモジュールはありませんが、このように命名することでコンパイラに`common`モジュールを結合テストファイルとして扱わないように指示します。

`tests/common/mod.rs`からモジュールを使用するには以下のように記述します。

```rust
extern crate testing;

mod common;

#[test]
fn it_adds_two() {
    common::setup();
    assert_eq!(4, testing::add_two(2));
}
```

**バイナリクレート**

もしプロジェクトが`src/main.rs`ファイルのみを含み、`src/lib.rs`ファイルを持たないバイナリクレートだったとします。
となると`tests`ディレクトリに結合テストを作成し、`extern crate`を使用して`src/main.rs`ファイルに定義された関数をインポートできなくなります。
ライブラリクレートのみが、他のクレートが呼び出して使用できる関数を晒せるのです。

これは、バイナリを提供するRustプロジェクトに、`src/lib.rs`ファイルに存在するロジックを呼び出す単純な`src/main.rs`ファイルがある一因になっています。
この構造を使用して結合テストは、`extern crate`を使用して重要な機能を用いることで、ライブラリクレートをテストすることができます。
この重要な機能が動作すれば、`src/main.rs`ファイルの少量のコードも動作し、その少量のコードはテストする必要がなくなるわけです。

## まとめ

Rustのテスト機能は、変更を加えた後でさえ想定通りにコードが機能し続けることを保証し、コードが機能すべき方法を指定する手段を提供します。
単体テストは、ライブラリの異なる部分を個別に用い、非公開の実装詳細をテストすることができます。
結合テストは、ライブラリの色々な部分が共同で正常に動作することを確認し、ライブラリの公開APIを使用して外部コードが使用するのと同じ方法でコードをテストします。
Rustの型システムと所有権ルールにより防がれるバグの種類もあるものの、それでもテストはコードが振る舞うと予想される方法に関するロジックのバグを減らすのに重要なのです。
