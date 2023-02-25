# パニックになるかパニックにならないか

> Ref: https://doc.rust-lang.org/stable/book/ch09-03-to-panic-or-not-to-panic.html

いつ`panic!`を呼び出すべきか、いつ`Result`を返すべきかをどのように判断すればいいのでしょうか。
コードがパニックになると、回復する方法がなくなります。回復可能な方法があるかどうかに関わらず、あらゆるエラー状況に対して`panic!`は呼び出すことが出来ます。
その場合、呼び出し元のコードに変わって回復不可能な状況であるという判断を下すことになります。
`Result`値を返すことにした場合、呼び出し側のコードにオプションを与えることになります。
呼び出し元のコードは、その状況に適した方法で回復を試みたり、この場合のErr値は回復不可能であると判断して、`panic!`を呼び出し、回復可能なエラーを回復不可能なエラーに変えることもできます。
したがって、失敗する可能性のある関数を定義する場合、`Result`を返すことは良いデフォルトの選択となります。

例題、プロトタイプコード、テストなどの状況では、`Result`を返す代わりに`panic!`を起こすコードを書く方が適切です。
その理由を説明した後、コンパイラは失敗が判断はできないが、人間なら判断できる状況について説明します。
この章の最後では、ライブラリコードでパニックを起こすかどうかを決定するための一般的なガイドラインを示します。

## 例、プロトタイプコード、テスト

あるコンセプトを説明するために例を書いている時、堅牢なエラー処理コードも含めると、例がわかりにくくなることがあります。
例えば、`unwrap`のようなパニックを起こす可能性のあるメソッドの呼び出しは、アプリケーションがエラーを処理する方法のプレースホルダーとして意図されていることが理解されます。

同様に、`unwrap, expect`メソッドは、プロトタイピングの際にエラーの処理方法を決定する前に非常に便利です。
これらはプログラムをより堅牢にする準備ができた時のために、コードに明確な目印を残しておくのです。

テストの中でメソッドの呼び出しに失敗したら、たとえそのメソッドがテスト対象の機能でなかったとしても、テスト全体を失敗させたいと思うことでしょう。
`panic!`はテストが失敗したとマークされる方法なので、`unwrap, expect`をコールすることはまさに怒るべきことなのです。

## コンパイラよりの多くの情報を持っている場合

また、`unwrap, expect`を呼び出すのは、`Result`が`Ok`の値を持つことを保証する他のロジックがある場合に適切です。
しかしそのロジックはコンパイラが理解できるものではありません。自分で呼び出した操作は、自分の特定の状況では論理的に不可能であっても、一般的にはまだ失敗する可能性があるのです。
もしあなたが手動でコードを検査することによって、決して`Err`変種を持たないことを確実にできるなら、`unwrap`を呼ぶことは完全に受け入れられるし、期待するテキストで`Err`変種を決して持たないと思う理由を文書化することはさらに良いことです。

```rust
use std::net::IpAddr;

let home: IpAddr = "127.0.0.1"
    .parse()
    .expect("Hardcoded IP address should be valid");
```

ハードコードされた文字列をパースして`IpAddr`インスタンスを作成しています。127.0.0.1は有効なIPアドレスであることがわかるので、ここで`expect`を使用しても構いません。
しかし、ハードコードされた有効な文字列があっても、`parse`メソッドの戻り値の型は変わりません。
`Result`値を取得することに変わりなく、コンパイラは`Err`変数の可能性があるかのように`Result`を処理させるでしょう。
もしIPアドレスの文字列がプログラムにハードコードされているのではなく、ユーザーから送られてきたもので、そのため失敗する可能性があるとしたら、もっと堅牢な方法で`Result`を処理したいと思うはずです。
このIPアドレスを他のソースから取得する必要がある場合、より良いエラー処理コードに期待値を変更するよう促しています。

## エラーハンドルのガイドライン

コードが悪い状態になる可能性がある場合、コードパニックを起こすことが望ましいです。
悪い状態というのは、何らかの前提、保証、契約、普遍量が破られた状態のことです。
例えば無効な値、矛盾する値、欠落した値がコードに渡された時、さらに以下のうち1つ以上が起きた場合のことも指します。

- ユーザーが間違ったフォーマットでデータを入力する
- 使用する型にエンコードする良い方法がない

誰かがあなたのコードを呼び出して、意味のない値を渡した場合、可能であればエラーを返し、ライブラリのユーザーがその場合にどうしたいかを決めることができるようにするのが一番です。
しかし継続が安全でない、あるいは有害である可能性がある場合、最善の選択は`panic!`を呼び出し、ライブラリを使っている人にコードのバグを警告し、開発中にそれを修正できるようにすることかもしれません。
同様に、自分では制御できない外部のコードを呼び出していて、そのコードが無効な状態を返し、それを修正する方法がない場合にも`panic!`を呼び出した方が良いでしょう。

しかし失敗が予想される場合は、`panic!`を呼び出すより`Result`を返す方が適切です。
例えば、パーサーに不正なデータが渡された場合や、HTTPリクエストがレートリミットを超えたことを示すステータスを返した場合などです。
このような場合、`Result`を返すことは失敗が予想され、呼び出し側のコードがその対処方法を決定しなければならないことを示します。

無効な値を使って呼び出された場合に、ユーザーを危険に晒す可能性のある操作を行う場合、コードはまず値が有効であることを確認し、値が有効でない場合パニックにすべきです。
これは主に安全上の理由から無効なデータに対して操作を行おうとすると、コードが脆弱性に晒される可能性があります。
これは標準ライブラリが境界外メモリアクセスを試行すると`panic!`関数にはコントラクトがあり、入力が特定の条件を満たす場合にのみその動作が保証されます。
なぜならコントラクト違反は常に呼び出し側のバグを示し、呼び出し側のコードが明示的に処理しなければならないような種類のエラーではないからです。
実際、呼び出し側のコードが回復する合理的な方法はありません。呼び出し側のプログラマはコードを修正する必要があります。
特に違反するとパニックになるような関数の契約は、その関数のAPIドキュメントで説明されるべきです。

しかし全ての関数にたくさんのエラーチェックがあると、冗長でイライラします💢
幸いRustの型システム（型チェック）を利用すれば、多くのチェックを代わりに行なってくれます。
関数が特定の型をパラメータとして持っている場合、コンパイラがすでに有効な値を持っていることを保証しているので、コードのロジックを進めることが出来ます。
例えば、`Option`ではなく型を指定した場合、プログラムは何もないのではなく、何かを持っていることを期待します。
この場合、コードは`Some, None`の2つのケースを処理する必要がありません。関数に何でも渡そうとするコードはコンパイルすらできないので、関数は実行時にそのケースをチェックする必要はありません。
他の例として、`u32`のような符号なし整数型を使用すると、パラメータが決して負にならないことを保証します。

