ActiveSupport Core Ext
====


### テーマ

- ```Object#try```   
       
        
**`#`で書いてるのはインスタンスメソッドなため**

### よく使うパターン

```
def activity(name)
  User.find_by(name: name).try(:state) || 'none'
end
```

### ソースコード

```
class Object
  def try(*a, &b) 
    if a.empty? && block_given?
      yield self
    else
      public_send(*a, &b) if respond_to?(a.first)
    end 
  end 
end

class NilClass
  def try(*args)
    nil
  end

  def try!(*args)
    nil
  end
end
```

### キーワード

- 引数
- `respond_to?`
- `block_given?`, `yield`
- `public_send`

### 引数の渡し方

- アスタ付きは配列を受け取る

```

```

- &を付けるとブロックを受け取る

```

```

###  respond_to? && public_send

- メソッドの存在確認
- パブリックメソッドの呼び出し

### yield

- ブロックを受け取る
- block_given?でblockがあったら、yieldでブロックを実行する。


ブロックって何？
----

### 身近なブロック

```
[1, 2, 3].each do |n|
  puts n
end
=> 1 2 3 
```

- `do ~ end`がブロックにあたる

### ブロックの使い道

```
A => B => C
```

AとCはこちらでセッティングするから、Bだけ他の人に決めてもらいたい時

###  日本語訳

```
[1, 2, 3].map{|n| n * 2}
=> 2, 4, 6

[1, 2, 3].select{|n| n % 2 == 0}
=> [2]
```

- map
  - 「配列を下さい。」
  - 「一つ一つに対する処理を書いて下さい」
  - 「結果を配列にして返します」
  
- select
  - 「配列を下さい」 
  - 「ある条件を書いて下さい」
  - 「結果を配列にして返します」
  
**真ん中の処理をユーザに任せている**  

###  