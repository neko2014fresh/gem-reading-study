はじめに
===

### 本日の流れ

- Gemの概要
- Gemを読む手段
- Tips in SettingsLogic
- ここ分からへんやった
- ここ分からへんやったので教えてください
- おまけ


Settingslogicの概要
===


### 環境毎に定数を定義できるGem

- `app/models/constants.rb`

```
class Constants < Settingslogic
  source "#{Rails.root}/config/constants.yml"
  namespace Rails.env
end
```

- `config/constants.yml`

```
defaults: &defaults
  name: 'ASKA'
  address: 'Minami-Aoyama'

development:
  name: 'CHAGE'
  address: 'Fukuoka'

test:
  <<: *defaultsa

production:
  <<: *defaults
```

- 実行してみる

```
(main)> Constants.address

#=>Fukuoka
```


### ほんとに環境ごとに違うの？


```
$ rails c
# => development

$ RAILS_ENV=production rails c
# => production
```


Gemを読むコツ
===
    
### 何通りか方法がある

- コメントから読む（大城さん談）
- エントリポイントのアタリを付けてprint, pryを挟みまくる(動的解析)
- コミットログから読む

> カーネルハッカー・小崎資広の「コードを読む技術」 | サイボウズ式
http://cybozushiki.cybozu.co.jp/articles/m000316.html

>
"本は前から読むように最適化されるんだけど、それは著者がそうなるように頑張ってるからなんです。生の情報は最適化されていないんです。だから、ソースコードを本のように読むのは、かなり非効率な読み方なんです。"

### Gemを読む準備

-  clone

```
$ git clone git@github.com:neko2014fresh/gem-reading-study.git
$ cd gem-reading-study
```

- SettingsLogic

```
$ gem which settingslogic
rbenvdir/versions/2.0.0-p247/lib/ruby/gems/2.0.0/gems/settingslogic-2.0.9/

$ subl rbenvdir/versions/2.0.0-p247/lib/ruby/gems/2.0.0/gems/settingslogic-2.0.9/
```

**$EDITORを設定しておくと、bundle openで開けて便利**

### pry tips


- ls 
  - インスタンスメソッド・クラスメソッド
- show-source(中のソース)


Tips in Settingslogic
===

### instance_eval


- サンプル

```
class Foo
  def initailize
	@name = "bar"
  end
end

foo = Foo.new

foo.instance_eval do
  puts @name
end
#=> bar

foo.name
#=>NoMethodError: undefined method `name' for #<Foo:0x007fb57c838f68 @name="bar">
from (pry):10:in `__pry__'
```

- `class_eval` => 一般的なクラスに対するメソッドの動的定義
- `instance_eval` => 特異メソッドの動的定義



### setter method =>　`[] []=`

- イベントドリブン（？)のような動きをする

```
class Foo
  def [](key)
	p "[] method is called"
  end
  
  def []=(key, value)
	p "key => #{key}, value=> #{value}"
  end
end

foo = Foo.new
foo[3] = "Mike"
#=>"key => 3, value=> Mike"

foo[4]
#=> "[] method is called"
```

**で、これどう使われてるん？**


### SettingsLogicを追いかける

- `rails console`から起動する

```
[1] pry(main)> Constants

From: /Users/fjwr38/github/gem-reading-study/app/app/models/constants.rb @ line 2 :

	1: class Constants < Settingslogic
 => 2:   binding.pry
	3:   source "#{Rails.root}/config/constants.yml"
	4:   namespace Rails.env
	5: end
```

### スタックトレース


```
(main)> Constants.address
=>

(main)> Settingslogic.method_missing
=>

(main)> Settingslogic.instance
=>

(main)> Settingslogic#initialize
=>
....
```

- 勘なんだけど、多分`initialize`まで呼ばれれば機能するのが、このGemだと思う。


### 何でRails Envで呼び出す変数を変えることが出来るのか？


- `Settingslogic.initialize`に書いてる
- initializeに`binding.pry`を挟んでみる。

何かクラスメソッドとインスタンスメソッドの重複多くない？
===

### SettingsLogicのインスタンスの同じ名前のメソッドに渡しているだけ

```
Settingslogic.method_missing
Settingslogic#method_missing

Settingslogic.create_accessor_for
Settingslogic#create_accessor_for
```


### create_accessor_for(1/2)


- `Settingslogic.create_accessor_for`

```
def create_accessor_for(key)
  return unless key.to_s =~ /^\w+$/  # could have "some-setting:" which blows up eval
  Rails.logger.debug "classmethod...instance_eval...#{key}"
  instance_eval "def #{key}; instance.send(:#{key}); end"
end 
```

- `Settingslogic#create_accessor_for`

```
def create_accessor_for(key, val=nil)
  return unless key.to_s =~ /^\w+$/  # could have "some-setting:" which blows up eval
  instance_variable_set("@#{key}", val)
  self.class.class_eval <<-EndEval
	def #{key}
	  return @#{key} if @#{key}
	  return missing_key("Missing setting '#{key}' in #{@section}") unless has_key? '#{key}'
	  value = fetch('#{key}')
	  @#{key} = if value.is_a?(Hash)
		self.class.new(value, "'#{key}' section in #{@section}")
	  elsif value.is_a?(Array) && value.all?{|v| v.is_a? Hash}
		value.map{|v| self.class.new(v)}
	   else
			value
	  end
	end
  EndEval
end
```

### create_accessor_for(2/2)


- 今回の場合このようなメソッドが定義される

```
class Constants
  # Constants.create_accessor_forで生成される  
  def self.address
    instance.send(:address)
  end
  
  # Constants#create_accessor_forで生成される
  def address
    return @key if @key
    value = fetch("#{key}")
    value
  end
end
```

結局分からなかった
===

### 教えてください

- これとか何に使ってんの？

```
def [](key)
  instance.fetch key.to_S, nil
end
```

```
def []=(key, val) 
  val = new(val, source)
end
```

- 何で`instance_eval`と `class_eval`が別にしてんの？両方`class_eval`でよくね？

おまけ(しょぼいTips)
===

### bundle open(1/2)



```
$ cd some/rails/project
$ bundle open #{gem_name}
```

- `~/.bashrc`

```
export EDITOR=vim
```

### bundle open(2/2)


```
$ mkdir ~/bin
$ ln -s "/Applications/Sublime Text 2.app/Contents/SharedSupport/bin/subl" ~/bin/subl

# .bashrcに以下のパスを追加

export PATH=$PATH:~/bin
```

### スタックトレースを追う


**pry-stack_explorer**

- 露骨なインターフェース（クラスメソッド）が提供されてるならまだしも、今回のようなどこがエントリポイントなのかよく分からない奴には多少有効かも？

```
gem 'pry-stack_explorer' 
```

- 適当な所でするとおｋ

```
(main) > show-stack
```

- 結構上手くいかないことがあるのでめげない。


Reference
===

>Ruby中級入門	
http://www.slideshare.net/shokai/ruby-24925828	
	
>ruby - How to understand the difference between class_eval() and instance_eval()? - Stack Overflow
http://stackoverflow.com/questions/900419/how-to-understand-the-difference-between-class-eval-and-instance-eval
   
>大城さん

次回予告
===

>ActiveSupport Core Extension   
http://edgeguides.rubyonrails.org/active_support_core_extensions.html

```
(main) > hoge.try(:id)
```

- 今年の間に`capistrano`とか読めたらｲｲﾅｧ
- 一人でやると心折れるので誰か一緒にやりません？