# Diary

## 2018/11/18 (sun)
ABC113のCをとちゅうで放り投げていたので、もう一度トライするも、自力ACを諦めた

市の情報を最後にどう反映すればいいのか悩んだ  
自分で考えた方法は残念ながらTLE  
なんかうまくstd::collectionsあたりを使って最終的な市の順番を取れないか  
考えてみたが、結局N^2な解放どまりだった。  


## 2018/11/17 (sat)
[これ](https://github.com/ko1/rubyhackchallenge/blob/master/JA/3_practice.md)をやる  
minirubyとかで簡単に動きが確認できるのが楽しい  

## 2018/11/11 (sun)

現在、
[The Rust Programming Language](https://doc.rust-lang.org/book/2018-edition/ch07-02-controlling-visibility-with-pub.html)
の7.2を読んでいる
privacyのruleを理解するためにサンプルを動かしてどうにか理解しようとしているところ
ruleの記述はいたってシンプル

```
If an item is public, it can be accessed through any of its parent modules.
If an item is private, it can be accessed only by its immediate parent module and any of the parent’s child modules.
```
この2ruleだが、そのルールのexampleが理解できなかった

```rust
cargo new hoge --lib

mod outermost {
    pub fn middle_function() {}

    fn middle_secret_function() {}

    mod inside {
        pub fn inner_function() {}

        fn secret_function() {}
    }
}

fn try_me() {
    outermost::middle_function();
    outermost::middle_secret_function();
    outermost::inside::inner_function();
    outermost::inside::secret_function();
}
```

これをコンパイルすると

```
~/hetak/rust-book/privacy-check $ cargo build
   Compiling privacy-check v0.1.0 (file:///mnt/c/Users/hetak/rust-book/privacy-check)
error[E0603]: function `middle_secret_function` is private
  --> src/lib.rs:14:5
   |
14 |     outermost::middle_secret_function();
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error[E0603]: module `inside` is private
  --> src/lib.rs:15:5
   |
15 |     outermost::inside::inner_function();
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error[E0603]: module `inside` is private
  --> src/lib.rs:16:5
   |
16 |     outermost::inside::secret_function();
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to 3 previous errors

For more information about this error, try `rustc --explain E0603`.
error: Could not compile `privacy-check`.

To learn more, run the command again with --verbose.
```

となるが
14行目で起きるコンパイルエラーが理解できなかったが、それはruleの
`immediate parent`がfunctionが存在するmoduleそのものを表していることに気が付けなかったからだった
(自分はもう一つ上のmoduleをimmediate parentだと思っていた)

ruleの2番目の後半`any of the parent’s child modules.`は、子供のmoduleのfnからだったらアクセスできる、ということだった
以下のようなことはOK

```rust
mod outermost {

   pub fn middle_function(){}

   fn middle_secret_function() {}

   mod inside {

      pub fn inner_function() {}

      fn secret_function() {}

      mod more_inside {

          // 2階層上のfnにもアクセス可能。ていうかsuper::で上に上がれることは正しいのか
          // どうかまだドキュメントに出てきていないので分からないが、とりあえずコンパイルは通った
          fn hoge() { super::super::middle_secret_function(); }

      }

   }


}
```

結局、ruleの記述にある`parent`の意味を誤解していただけだった、というオチでした
