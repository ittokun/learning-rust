# ライフタイム

> Ref: https://doc.rust-jp.rs/book-ja/ch10-03-lifetime-syntax.html

第4章で紹介しなかった1つに、Rustにおいて参照はすべてライフタイムを保持するというものがあります。
ライフタイムとは、その参照が有効になるスコープのことです。
多くの場合、型が推論されるようにライフタイムも暗黙的に推論されます。
複数の方の可能性があるときは型を注釈しなければなりません。
同様に参照のライフタイムがいくつかの異なる方法で関係することがある場合には注釈しなければなりません。
コンパイラは、ジェネリックライフタイム引数を使用して関係を注釈し、実行時に実際の参照が有効である保証をすることを要求します。

ライフタイムの概念は、他のプログラミング言語とは異なり、間違いなくRustで一番際立った機能になっています。
この章では、ライフタイム記法が必要となる最も一般的な場面について説明していきます。

## ライフタイムでダングリング参照を回避

ライフタイムの主な目的は、ダングリング参照を回避することです。
ダングリング参照によりプログラムは、参照するつもりだったデータ以外のデータを参照してしまいます。

```rust
{
    let r;

    {
        let x = 5;
        r = &x;
    }

    println!("r: {}", r);
}
```

> `let r`の行では変数に初期値を与えずに宣言しておりエラーが起きるように見えますがそうはなりません。
> しかし、値を与える前に変数をしようとするとコンパイルエラーとなります。

このコードはコンパイルできず、次のエラーメッセージを表示します。

```bash
error[E0597]: `x` does not live long enough
  --> src/main.rs:8:17
   |
8  |             r = &x;
   |                 ^^ borrowed value does not live long enough
9  |         }
   |         - `x` dropped here while still borrowed
10 |
11 |         println!("r: {}", r);
   |                           - borrow later used here

For more information about this error, try `rustc --explain E0597`.
```

変数`x`の生存期間が短すぎますと書かれています。これは`r`が`x`を参照する前に`x`がスコープから抜けているため、解放されたメモリを`r`が参照しようとしているということです。
コンパイラは借用チェッカーを使用して、このコードが無効であると決定しています。

## 借用チェッカー

Rustコンパイラには、スコープを比較してすべての借用が有効であるかを決定する借用チェッカーがあります。

```rust
{
    let r;                // --------+-- 'a
                          //         |
    {                     //         |
        let x = 5;        // -+-- 'b |
        r = &x;           //  |      |
    }                     // -+      |
                          //         |
    println!("r: {}", r); //         |
}                         // --------+
```

変数`x`を変数`r`と同じスコープで定義するとエラーなくコンパイルできます。

```rust
{
    let x = 5;            // ---------+-- 'b
                          //          |
    let r = &x;           // --+-- 'a |
                          //   |      |
    println!("r: {}", r); //   |      |
                          // --+      |
}                         // ----------
```

これがコンパイラがライフタイムを解析して参照が常に有効であることを保証する仕組みです。
次は、関数における引数と戻り値のジェネリックなライフタイムを見ていきます。

## 関数のジェネリックなライフタイム

例として2つの文字列スライスのうち、長い方を返す`longest`関数を書いていきます。
この関数は、2つの文字列スライスを引数として取り、1つの文字列スライスを返します。
`longest`関数の実装を完了すると、`The longest string is abcd`と出力するようにしていきます。

```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}
```

関数にとって欲しい引数が文字列スライスです。これは`longest`関数に引数の所有権を奪ってほしくないときに指定します。

以下のコードはコンパイルできません。

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

また次のようなエラ〜メッセージとヘルプが表示されます。

```bash
error[E0106]: missing lifetime specifier
  --> src/main.rs:14:37
   |
14 |     fn longest(x: &str, y: &str) -> &str {
   |                   ----     ----     ^ expected named lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
   |
14 |     fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
   |               ++++     ++          ++          ++

For more information about this error, try `rustc --explain E0106`.
```

ここでは、この関数の戻り値型は借用された値を含んでおりシグネチャは、それが`x, y`のどちらから借用されたものなのか宣言していないことを言っています。
つまり、戻り値の型はジェネリックなライフタイム引数である必要があるということです。

このエラーを修正するためには、借用チェッカーが解析を実行できるように、参照間の関係を定義するジェネリックなライフタイム引数を追加します。

## ライフタイム注釈記法

ライフタイム注釈は、いかなる参照の生存期間も変えることはありません。
ジェネリックなライフタイム引数を指定された関数は、あらゆるライフタイムの参照を受け取ることが出来ます。
ライフタイムの注釈は、ライフタイムに影響することなく、複数の3章のライフタイムのお互いの関係を記述します。

ライフタイム注釈は、少し不自然な記法です。
まずライフタイム引数の名前はアポストロフィー(`'`)で始め、名前は全部小文字で指定します。多くの場合、`'a`という名前を使用します。
ライフタイム引数注釈は、参照の`&`の後に配置し、注釈と参照の方を区別するために風伯を1つ使用します。

例を上から1つずつ見てみましょう。

1. ただの参照
2. 明示的なライフタイム付きの参照
3. 明示的なライフタイム付きの可変参照

```rust
&i32

&'a i32

&'a mut i32
```

## 関数シグネチャのライフタイム注釈

`longest`関数を例に、ライフタイム注釈を詳しくみていきましょう。
ジェネリックな型引数同様、関数名と引数の間の山カッコの中にジェネリックなライフタイム引数を宣言します。
このシグネチャで表現したい制約は、引数の全ての参照と戻り値が同じライフタイムを持つことです。

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

このコードはコンパイルできます。そして`main`関数内でも正常に動作します。

これで関数シグネチャは、何らかのライフタイム`'a`に対して、関数は2つの引数を取り、どちらも少なくともライフタイム`'a`と同じだけ生きる文字列スライスであるとコンパイラに教えるようになりました。
また戻り値にも同じように伝えることが出来ています。
実際には、`longest`関数が返す三章のライフタイムは、渡された参照のうち、小さい方のライフタイムと同じであるということです。
これらの制約はまさに、コンパイラに保証して欲しかったものなのです。

この関数シグネチャでライフタイム引数を指定するとき、渡されたり、返したり、いかなる値のライフタイムも変更していません。
むしろ借用チェッカーは、これらの制約を守らない値全てを拒否するべきと指定しています。
`longest`関数は、`x, y`の正確な生存期間を知っている必要はなく、このシグネチャを満たすようなスコープを`'a`に代入できることを知っているだけです。

関数にライフタイムを注釈するときは、注釈は関数の本体ではなくシグネチャに付与します。
コンパイラは注釈がなくとも関数内のコードを解析できます。
しかしながら、関数に関数外からの参照や関数外への参照がある場合、コンパイラが引数や戻り値のライフタイムを自力で解決することはほとんどないです。
そのライフタイムは、関数が呼び出されるたびに異なる可能性があり、手動でライフタイムを注釈する必要が生じます。

具体的な参照を`largest`に渡すと、`'a`に代入される具体的なライフタイムは、`x`のスコープの一部であって`y`のスコープと重なる部分となります。
言い換えると、ジェネリックなライフタイム`'a`は、`x, y`のライフタイムのうち、小さい方に等しい具体的なライフタイムになります。
返却される参照を同じライフタイム引数`'a`で注釈したので、返却される参照も`x, y`のどちらかのライフタイムの小さい方と同じだけ有効になります。

ライフタイム注釈が異なる具体的なライフタイムを持つ参照を渡すことで`longest`関数を制限する方法を見てみましょう。

```rust
fn main() {
    let string1 = String::from("long string is long");

    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    }
}
```

このコードは動作します。次に、`result`の三章のライフタイムが2つの引数の小さい方のライフタイムになることを示す例を見てみましょう。

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

このコードは動作しません。以下のようなエラーメッセージが表示されます。

```bash
error[E0597]: `string2` does not live long enough
  --> src/main.rs:37:48
   |
37 |             result = longest(string1.as_str(), string2.as_str());
   |                                                ^^^^^^^^^^^^^^^^ borrowed value does not live long enough
38 |         }
   |         - `string2` dropped here while still borrowed
39 |         println!("The longest string is {}", result);
   |                                              ------ borrow later used here

For more information about this error, try `rustc --explain E0597`.
```

このエラーは、`result`が`pringln!`に対して有効であるためには、`string2`が外側のスコープの終わりまで有効である必要があると書かれています。
関数引数と戻り値のライフタイムを同じライフタイム引数`'a`で注釈したので、コンパイラはこのことを知っています。

コンパイラは、関数に渡した参照のうち小さい方のライフタイムを評価するので、借用チェッカーは上のコードを無効な参照がある可能性があるとして許可しないのです。

## ライフタイムの観点で考える

何にライフタイム引数を指定する必要があるかは、関数が行なっていることに依存します。
例えば、`longest`関数の実装を最長の文字列スライスではなく、常に最初の引数を返すように変更したら、`y`引数に対してライフタイムを指定する必要はなくなります。

```rust
fn longest<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

関数から参照を返す際、戻り値型のライフタイム引数は、引数のうちどれかのライフタイム引数と一致する必要があります。
返される参照が引数のどれかを参照していないなら、この関数内で生成された値を参照しているはずです。
するとその値は関数の末端でスコープを抜けるので、これはダングリング参照になるのです。

```rust
fn longest<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("really long string");
    result.as_str()
}
```

上記のコードは戻り値型にライフタイム引数`'a`を指定しているので、これと関係している返り血にしなければいけません。
しかし指定された戻り値には引数のライフタイムと全く関係がないので、エラーとなります。

```bash
error[E0515]: cannot return reference to local variable `result`
  --> src/main.rs:45:9
   |
45 |         result.as_str()
   |         ^^^^^^^^^^^^^^^ returns a reference to data owned by the current function

For more information about this error, try `rustc --explain E0515`.
```

問題は、`result`が`longest`関数の末端でスコープを抜け、片付けられてしまうことです。かつ、関数から`result`への参照を返そうともしています。
ダングリング参照を変えてくれるようなライフタイム引数を指定する手段はなく、コンパイラはダングリング参照を生成させてくれません。
今回の場合の最善の修正案は、呼び出し元の関数に値の片づけをさせるために、参照ではなく所有されたデータ型を返すことでしょう。

究極的にライフタイム記法は、関数の色々な引数と戻り値のライフタイムを接続することに関するものです。
一旦それがつながれば、メモリは安全な処理を許可し、ダングリングポインタを生成したりメモリ安全性を侵害したりする処理を禁止するのに十分な情報をコンパイラ映えたことになります。

## 構造体定義のライフタイム注釈

ここまでは所有された方を保持する構造体だけを定義してきました。
構造体に参照を保持させることもできますがその場合、構造体定義の全参照にライフタイム注釈を付け加える必要があります。

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
    println!("Important Excerpt: {}", i.part);
}
```

この構造体には文字列スライスを保持する`part`フィールドという参照があります。
ジェネリックなデータ型同様、構造体名の後の山カッコの中にジェネリックなライフタイム引数の名前を宣言することで、構造体定義の本体でライフタイム引数を使用できます。
この注釈は、`ImportantExcerpt`のインスタンスが、`part`フィールドに保持している山椒よりも長生きしないことを意味します。

## ライフタイム省略

全参照にはライフタイムがあり、参照を使用する関数や構造体にはライフタイム引数を指定する必要があります。
しかし以下のコードはライフタイム注釈なしでコンパイルすることが出来ます。

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

この関数がライフタイム注釈なしでコンパイルできるのは、Rustの歴史が関わっています。
Rust:1.0以前では、全参照に明示的なライフタイムが必要だったので、このコードはコンパイルできませんでした。
その頃の関数シグネチャはこのように記述されていました。

```rust
fn first_word<'a>(s: &'a str) -> &'a str {}
```

多くのRustコードを書いた後、RustチームはRustプログラマが特定の場所で何度も同じライフタイム注釈を入力していることを発見しました。
これらの場面は予測可能で、いくつかの決まり切ったパターンに従っていました。
開発者はこのパターンをコンパイラのコードに落とし込んだので、このような場面には借用チェッカーがライフタイムの推論できるようになり、明示的な注釈を必要としなくなったのです。

ここでRustの歴史話をしたのは、将来的にさらに少数のライフタイム注釈しか必要にならない可能性もあるということを伝えるためです。

コンパイラの参照解析に落とし込まれたパターンは、ライフタイム省略規則と呼ばれます。
これらはプログラマが従う規則ではなく、コンパイラが考慮する一連の特定のケースであり、このケースに当てはまれば、ライフタイムを明示的に書く必要はなくなります。

省略規則は完全な推論を提供しません。コンパイラが結果的に規則を適用できるが、参照を保持するライフタイムに関して曖昧性があるならライフタイムを推測しません。
この場合コンパイラは、それらを推測するのではなくエラーを与えます。

関数やメソッドの引数のライフタイムは入力ライフタイムと呼ばれ、戻り値のライフタイムは出力ライフタイムと称されます。

コンパイラは3つの規則を活用し、明示的な注釈がないときに、3章がどんなライフタイムになるか計算します。
最初の規則は入力ライフタイムに適用され、2、3番目の規則は出力ライフタイムに適用されます。
コンパイラが3つの規則の最後まで到達し、それでもライフタイムを割り出せない参照があれば、コンパイラはエラーを吐きます。
これらのルールは`fn, impl`にも適用されます。

最初の規則は、参照である各引数に独自のライフタイム引数を得るというものです。
要は、1引数の関数は、1つのライフタイム引数を得るというものです。
1つのライフタイムの場合、`fn foo<'a>(x: &'a i32)`
2つのライフタイムの場合、`fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`

2番目の規則は、1つだけ入力ライフタイム引数があれば、そのライフタイムがすべての出力ライフタイム引数に代入されるというものです。例：`fn foo<'a>(x: &'a i32) -> &'a i32`

3番目の規則は、複数の入力ライフタイム引数があるメソッドでそのうちの1つが`&self, &mut self`であれば、`self`のライフタイムが全出力ライフタイム引数に代入されるというものもです。
これにより必要なシンボルの数が減るので、メソッドがはるかに読み書きしやすくなります。

コンパイラの立場になってみましょう。これらの規則を適用して、`first_word`関数のシグネチャの参照のライフタイムが何か計算されます。
シグネチャは参照に紐付けられるライフタイムがない状態から始まります。

最初の規則では、各引数が独自のライフタイムを得ると指定します。

次に2番目の規則では、1つの入力引数のライフタイムが、出力引数に代入されます。

```rust
fn first_word(s: &str) -> &str {}

fn first_word<'a>(s: &'a str) -> &str {}

fn first_word<'a>(s: &'a str) -> &'a str {}
```

もうこの関数シグネチャの全ての参照にライフタイムがつけられたので、コンパイラはプログラマにライフタイムを注釈してもらう必要はなく、解析を続行できます。

次は`longest`関数で見ていきます。

2つ以上入力ライフタイムがあるので、2番目の規則は適用されません。また3番目の規則も適用されません。
`longest`はメソッドではなく関数なので、どの引数も`self`ではありません。
そのため3つの規則全部を適用した後でも、戻り値型のライフタイムが判明しません。

```rust
fn longest(x: &str, y: &str) -> &str {}

fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str {}
```

実際のところ3番目の規則はメソッドのシグネチャにしか適用されません。
なので次はその文脈においてライフタイムを観察し、3番目の規則のおかげで、メソッドシグニチャであまり頻繁にライフタイムを注釈しなくても済む理由を見ていきます。

## メソッド定義におけるライフタイム注釈

構造体にライフタイムのあるメソッドを実装する際、ジェネリックな型引数と同じ記法を使用します。
ライフタイム引数を宣言し使用する場所は、構造体フィールドかメソッド引数と戻り値に関係するかによります。

構造体フィールド用のライフタイム名は、`impl`キーワードの後に宣言する必要があり、それから構造体名の後に使用されます。

`impl`ブロック内のメソッドシグネチャでは、参照は構造体のフィールドの参照のライフタイムに紐づいている可能性と、独立している可能性があります。
加えてライフタイム省略規則により、メソッドシグネチャでライフタイム注釈が必要なくなることもあります。

```rust
impl<'a> ImportantExcerpt<'a> {
    fn lebel(&self) -> i32 {
        3
    }
}
```

また3番目のライフタイム省略規則が適用される例はこちらです。

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

2つの入力ライフタイムがあるので、コンパイラは最初のライフタイム省略規則を適用し、`&self, announcement`に独自のライフタイムを与えます。
それから引数の1つが`&self`なので、戻り値型は`&self`のライフタイムを得て、全てのライフタイムが説明されます。

## 静的ライフタイム

`'static`は、参照がプログラムの全期間生存することのできる特殊なライフタイムです。
文字列リテラルは全て`'static`になり次のように注釈できます。

```rust
let s: &'static str = "I have a static lifetime.";
```

この文字列のテキストは、プログラムのバイナリに直接格納され常に利用可能になります。
故に前文字列リテラルのライフタイムは`'static`となります。

エラーメッセージで、`'static`ライフタイムを使用するよう勧める提言を見ることがあります。
しかし参照に対して`'static`都指定する前に、今ある参照が本当にプログラムの全期間生きるかどうか考えてください。
それが可能であったとしても、参照がそれだけの期間生きて欲しいのかどうか考慮するのもいいでしょう。
ほとんどの場合、問題はダングリング参照を生成しようとしているか、利用可能なライフタイムの不一致が原因です。
そのような場合、解決策はその問題を修正することであり、`'static`を指定することではありません。

## ライフタイムを一度に指定

ジェネリックな型引数、トレイト境界、ライフタイム指定の構文すべてを1つの関数で指定する例を見てみましょう。

```rust
fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    println!("longest is: {}", longest_with_an_announcement("hoge", "bar", "baz"));
}
```

### まとめ

この章では、ジェネリックな型引数、トレイトとトレイト境界、ジェネリックなライフタイム引数を学びました。
これでコードを繰り返すことなく書く準備ができました。
- ジェネリックな型引数では、コードを異なる型に適用させてくれます。
- トレイトでは、型がジェネリックであっても、コードが必要とする振る舞いを持つことを保証します。
- ライフタイム注釈では、柔軟なコードにダングリング参照が存在しないことを保証します。

さらにこれらの解析はすべてコンパイル時に起こり、実行時のパフォーマンスには影響しません！

第17章では、この話題を深く掘り下げたトレイトオブジェクトについて説明します。
これはトレイトを使用する別の手段です。非常に高度な筋書きの場合でのみ必要で、ライフタイム注釈も関わるもっと複雑な筋書きもあります。

次の章では、コードがあるべき通りに動いているか確認できる、テストを書く方法を学びます。
