# Diary

## 2018/12/19 (wed)
[ledermann/unread](https://github.com/ledermann/unread)を調べた。
使い方とかは、基本的にはREADMEを見れば何となく分かるけど、ソース読んだりしたので少しメモっておく

- これは何？  
既読・未読機能を実現するためのgem  
インストール方法やメソッドの呼び出し例などは本家サイトのREADME.mdを参照


- 用語  
`readable` : 既読・未読対象となる主体のこと。ex. `Message`  
`reader` : 読んで字のごとく、既読・未読のアクションを実行する主体。ex. `User`  
globalなレコード : 既読、未読を判定する基準となるタイムスタンプが格納されているレコードのこと。`readable_id`カラムが`NULL`になっている。

- 準備  
`acts_as_reader` を対象のモデル内のクラスマクロとして呼び出す  
`acts_as_readable` を対象のモデル(ry  
特に`acts_as_readable`では、`on`オプションを使うと既読・未読の判定時に利用するタイムスタンプカラムを変更可(デフォルトは`updated_at`)

- DBテーブル  
`read_marks`  
`readable`と`reader`を紐付ける中間テーブルの役割を果たす

- データの初期化  
`lib/unread/reader.rb`の`setup_new_reader`で実施  
特に、`lib/unread/base.rb`の`act_as_reader`にて、`after_create :set_new_reader`していることに注意  
初期化時には`readable_id`に`NULL`が突っ込まれる  
本gemでは、`global`という単語が出てくるが、これはこの時に生成されたレコードのことを指す

- 未読 -> 既読  
`lib/readable`の`mark_as_read!`というクラスメソッド、インスタンスメソッド両方が存在し、それらを呼び出すことで既読扱いにする  
`mark_as_read!`は`unread?`を呼び出して、`true`ならば以下のようにして既読状態にする  

  ```ruby
  ReadMark.transaction do
    if unread?(reader)
      rm = read_mark(reader) || read_marks.build
      rm.reader_id   = reader.id
      rm.reader_type = reader.class.base_class.name
      rm.timestamp   = self.send(readable_options[:on])
      rm.save!
    end
  end
  ```

  `unread?`は`with_read_marks_for`メソッドの判定用のところを一旦無視すると  
    `self.class.unread_by(reader).exists?(self.id)`  
  のワンライナーで行っている  

- `unread_by`について  

  ```ruby
  def unread_by(reader)
    result = join_read_marks(reader)

    if global_time_stamp = reader.read_mark_global(self).try(:timestamp)
      result.where("#{ReadMark.quoted_table_name}.id IS NULL
                    AND #{quoted_table_name}.#{connection.quote_column_name(readable_options[:on])} > ?", global_time_stamp)
    else
      result.where("#{ReadMark.quoted_table_name}.id IS NULL")
    end
  end
  ```

  - `join_read_marks`は`readable`テーブルと`read_marks`テーブルとをleft joinして`global_time_stamp`よりもreadableテーブルのレコードのタイムスタンプが更新処理などによって若くなれば、未読判定となる。    
  - `"#{ReadMark.quoted_table_name}.id IS NULL`の部分がなぜ必要かというと、`join_read_marks`で`left join`しているので、そのSQLの取得結果は、未読扱いものは`read_marks.id`が`NULL`になるから  

  ```ruby
  def join_read_marks(reader)
    assert_reader(reader)

    joins "LEFT JOIN #{ReadMark.quoted_table_name}
            ON #{ReadMark.quoted_table_name}.readable_type  = '#{readable_parent.name}'
           AND #{ReadMark.quoted_table_name}.readable_id    = #{quoted_table_name}.#{quoted_primary_key}
           AND #{ReadMark.quoted_table_name}.reader_id      = #{quote_bound_value(reader.id)}
           AND #{ReadMark.quoted_table_name}.reader_type    = #{quote_bound_value(reader.class.base_class.name)}
           AND #{ReadMark.quoted_table_name}.timestamp     >= #{quoted_table_name}.#{connection.quote_column_name(readable_options[:on])}"
  end
  ```

- 既存データに対する対応(2018/12/20 追記)   
既存レコードに対して本gemを適用するには一つ注意が必要である  
`reader`となる既存レコードに対して、globalなレコードを作成する必要がある:  

  ```ruby
  User.find_each { |user| Comic.mark_as_read!(:all, for: user) }  
  ```

- クリーンアップ用メソッド `cleanup_read_marks!` について  
https://github.com/ledermann/unread#how-does-it-work   
既読レコードが増え続けるので、cronによるバッチ処理などで、このメソッドを呼び出す必要がある  
ex. `Message.cleanup_read_marks!`    
このメソッドの処理は大きく分けて2つのことを行う:  
  - すでに既読扱いになっているレコードの削除
  - globalなレコードのタイムスタンプのバッチ実施時刻への更新


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
