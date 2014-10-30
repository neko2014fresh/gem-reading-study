Reading Cocaine
====

### Gemを読むことばかりの人生ですみません

- でもネタが無いんです
     
- 家帰ると何もやる気がしないんです
    


最近の悩み
===

### ファイルシステムに関わることをすると

- 何かしらファイルシステムに対して変更を加える必要がある(quotaとか)
    
- それがrubyでAPI提供されてないことあり
  - Cでも提供されてない場合もあり
     
- そういうときにどうするか....


Unixコマンド叩いて結果をパースする
====

### これがまた泥臭い

- そもそもコマンドの実行に失敗することあり
- そういった時にうまく例外拾えないと結構な悲劇
- ファイルシステムにロールバックなんて便利なもんは無いんや
- cocaineでごまかしている

Cocaine is 何
====

### RubyからUnixコマンドを実行する

- 以下二つは同じ

```
$ id r-fujiwara
```

```
cmd = Cocaine::CommandLine.new("id", ":user_name")  
cmd.run user_name: 'r-fujiwara'
```

- 実際

```
cmd = Cocaine::CommandLine.new("sudo gluster", "volume quota :volume_name limit-usage :path_for_gluster :quota")
cmd.run volume_name: volume_name.to_s, path_for_gluster: path_for_gluster.to_s, quota: quota
```


### Cocaineのポイント

- どのくらいCocaine側で例外処理をカバーしてるのか？
- コマンドを安全に実行しているのは、どういう仕組みか？


大事なこと
=====

### 悲しい事実

- cocaine上での例外キャッチは**4ヵ所**
- `CommandNotFound`か**それ以外**しかない
- つまりCocaine使うときは例外処理必須です



###  ふわっとした流れ

```
def run(interpolations = {})
  output = ''
  @exit_status = nil
  begin
    # 整形
    full_command = command(interpolations)
    log("#{colored("Command")} :: #{full_command}")
    # 実行
    output = execute(full_command)
```

- 多分`command(interpolations)`でコマンドを整形して
- `execute(full_command)`で実行していると思われる

### initialize

```
    def initialize(binary, params = "", options = {})
      @binary            = binary.dup
      @params            = params.dup
      @options           = options.dup
      @runner            = @options.delete(:runner) || self.class.runner
      @logger            = @options.delete(:logger) || self.class.logger
      @swallow_stderr    = @options.delete(:swallow_stderr)
      @expected_outcodes = @options.delete(:expected_outcodes) || [0]
      @environment       = @options.delete(:environment) || {}
      @runner_options    = @options.delete(:runner_options) || {}
    end
```

- Cocaine::CommandLine.new("first", "second")
- この例で言うと`@binary`に`first`, `@params`に`second`が入ってくる

### 安全なコマンドを生成している部分

```
def command(interpolations = {})
  cmd = [@binary, interpolate(@params, interpolations)]
  cmd << bit_bucket if @swallow_stderr
  cmd.join(" ").strip
end
```

- めんどうなので解説しません

### コマンドを実行部分

```
def run
### 略
  output = execute(full_command)
#### 略
end

######

def execute
  runner.call(command, environment, runner_options)  
end
```

- とりあえず`runnner`には`ProcessControl`ってのが入ってくるので実行している

### ProcessRunnner

```
module Cocaine
  class CommandLine
    class ProcessRunnner
      def call
        input, output = IO.pipe
        options[:out] = output
        with_modified_environment(env) do
        pid = spawn(env, command, options)
        output.close
        result = input.read
        waitpid(pid)
        input.close
        result  
      end
    end
  end
end
```

コマンドの実行
====

### spawn

- OSのコマンドを実行してくれる

>RubyでOSの外部コマンドを実行したいときに、spawnを使うと便利だった件 - カイワレの大冒険 Second
http://www.masudak.net/blog/2013/05/30/201305302308/



### ClimateControl

- ブロックの中で環境変数を定義するもの
- 結構面白いのでは？


**ClimateControl can be used to temporarily assign environment variables within a block:**


```
ClimateControl.modify CONFIRMATION_INSTRUCTIONS_BCC: 'confirmation_bcc@example.com' do
  sign_up_as 'john@example.com'
  confirm_account_for_email 'john@example.com'
  current_email.should bcc_to('confirmation_bcc@example.com')
end
```

>
ClimateControl
https://github.com/thoughtbot/climate_control


言いたいこと
====

低意識(ダイソン)例外処理やめよう
====


### これではなく

```
begin
  some_func
# 何でもダイソンが如くキャッチする
rescue => e
  unsuitable_func
end
```

- とりあえずeでキャッチ

### これ

```
begin

# 想定される例外をキャッチ
rescue SomeHogeError => e
  suitable_func
rescue => e
# 最終的に分からないから
  unsuitable_func
end
```
