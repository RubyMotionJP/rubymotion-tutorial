---
layout: default
title: API Driven Example
full_title: An API-Driven Example
description: A complete RubyMotion example app using HTTP to access a remote API
categories:
- chapters
---

# APIドリブン例：Colr

私達は [Colr JSON API][colr] のフロントエンドを構築します．ユーザーは 16 進コード ( 例: "#3B5998") で色を入力することができ，その色に Colr ユーザーがどういったタグを割り当てているか見ることができます．もし特筆すべき新しいものだと感じた場合，新しいタグを追加できます！

大まかな設計について話しましょう．私達には 2 つのコントローラが必要です: 一つは検索のため，もう一つは色のため．それらはトップレベルのコントローラである `UINavigationController` の内部にラップされるべきです．`Color` や `Tag` といった，いくつかのモデルもまた必要になるでしょう．正直に言うと，私達のアプリは Dribbble の次のトップ記事にはならないでしょうけれども，まあとにかく動きはするでしょう．

## セットアップ

`motion create Colr` でプロジェクトのセットアップをし，`require 'bubble-wrap'` を `Rakefile` に追加します．適切な設定を行ったら，`./app` *内部*に いくつかのフォルダ - `./app/models/` 及び `./app/controllers` を作成します．

## モデル

まずモデルを掘り下げてみましょう．Colr API は JSON オブジェクトの形式で自身の色を返します :

```
{
  "timestamp": 1285886579,
  "hex": "ff00ff",
  "id": 3976,
  "tags": [{
    "timestamp": 1108110851,
    "id": 2583,
    "name": "fuchsia"
  }]
}
```

そこで，私達の `Color` には `timestamp`， `hex`， `id`，および `tags` のプロパティが必要になります．特に，`tags` は `Tag` オブジェクトの has-many 関係を表しています．

`./app/models/color.rb` を作成して，モデルのテンプレートを書き込んでください :

```ruby
class Color
  PROPERTIES = [:timestamp, :hex, :id, :tags]
  PROPERTIES.each { |prop|
    attr_accessor prop
  }

  def initialize(hash = {})
    hash.each { |key, value|
      if PROPERTIES.member? key.to_sym
        self.send((key.to_s + "=").to_s, value)
      end
    }
  end

  ...
```

`PROPERTIES` のトリックを使うのはとても簡単です．でも `tags` 属性についてだけ，常に `Tag` の配列であることという特別な前提を強制することにします :

```ruby
  ...

  def tags
    @tags ||= []
  end

  def tags=(tags)
    if tags.first.is_a? Hash
      tags = tags.collect { |tag| Tag.new(tag) }
    end

    tags.each { |tag|
      if not tag.is_a? Tag
        raise "Wrong class for attempted tag #{tag.inspect}"
      end
    }

    @tags = tags
  end
end
```

`#tags` getter は `tags` にまだ値が与えられていない場合でも配列を返すことを保証します．`#tags=` setter は，引数で渡された `tags` に含まれる全てのオブジェクトが `Tag` オブジェクトであることを保証します．でも，待って……まだ `Tag` クラスを書いていませんでした！

`./app/models/tag.rb` を作成して開きましょう．上記の例からわかるように，API から返されたタグの形式は次のとおりです :

```
{
  "timestamp": 1108110851,
  "id": 2583,
  "name": "fuchsia"
}
```

そこで，素晴しい API に馴染むようなモデルとなるように `Tag` クラスを作りましょう．特別なオーバーライドはありません．この状態で十分動作します :

```ruby
class Tag
  PROPERTIES = [:timestamp, :id, :name]
  PROPERTIES.each { |prop|
    attr_accessor prop
  }

  def initialize(hash = {})
    hash.each { |key, value|
      if PROPERTIES.member? key.to_sym
        self.send((key.to_s + "=").to_s, value)
      end
    }
  end
end
```

## コントローラー

今，私たちにはモデルがありますので，コントローラーを作り始めましょう．`./app/controllers/search_controller.rb` と `./app/controllers/color_controller.rb` を作ってください．さしあたって今は最低限の実装をすることにします :

```ruby
class SearchController < UIViewController
  def viewDidLoad
    super

    self.title = "Search"
  end
end
```

```ruby
class ColorController < UIViewController
  def viewDidLoad
    super

    self.title = "Color"
  end
end
```

そして `UIWindow` と `UINavigationController` を `AppDelegate` を使って画面上に表示します :

```ruby
class AppDelegate
  def application(application, didFinishLaunchingWithOptions:launchOptions)
    @window = UIWindow.alloc.initWithFrame(UIScreen.mainScreen.bounds)

    @search_controller = SearchController.alloc.initWithNibName(nil, bundle:nil)
    @navigation_controller = UINavigationController.alloc.initWithRootViewController(@search_controller)

    @window.rootViewController = @navigation_controller
    @window.makeKeyAndVisible
    true
  end
end
```

私たちは以前にもこのようなものを見たことがあり，一度だけではないかもしれません．`rake` して質素なアプリを見てみましょう :

![search title in app](images/0.png)

好調な出足ですね！`SearchController` を作り込んでいきましょう．

## SearchController

ユーザーから入力された 16 進数を検索するのに，`UITextField` という，今まで私たちが使ってこなかった新しいタイプの制御を使いましょう．ユーザが " 検索 " ボタンを押すと，適切な API リクエストを実行し，それが終了するまで UI をロックします．もし結果を見つけた場合，`ColorController` をプッシュします．そうではない場合，残念ながらアラートを表示します．よさそうな動きじゃないですか？

以下のようにして，ビューを `SearchController` の中にセットアップすることができます :

```ruby
  def viewDidLoad
    super

    self.title = "Search"

    self.view.backgroundColor = UIColor.whiteColor

    @text_field = UITextField.alloc.initWithFrame [[0,0], [160, 26]]
    @text_field.placeholder = "#abcabc"
    @text_field.textAlignment = UITextAlignmentCenter
    @text_field.autocapitalizationType = UITextAutocapitalizationTypeNone
    @text_field.borderStyle = UITextBorderStyleRoundedRect
    @text_field.center = CGPointMake(self.view.frame.size.width / 2, self.view.frame.size.height / 2 - 100)
    self.view.addSubview @text_field

    @search = UIButton.buttonWithType(UIButtonTypeRoundedRect)
    @search.setTitle("Search", forState:UIControlStateNormal)
    @search.setTitle("Loading", forState:UIControlStateDisabled)
    @search.sizeToFit
    @search.center = CGPointMake(self.view.frame.size.width / 2, @text_field.center.y + 40)
    self.view.addSubview @search
  end
```

沢山ある具体的なポジション設定 (`self.view.frame.size.height / 2 - 100` など ) は，私が勘で決めて動作確認しただけで，魔法のようなことは何もしていません（残念です）．初めてみる設定がいくつかあります．`UIControlStateDisabled` は，もしボタンが `@search.enabled = false` の場合にどのように見えるかを示しており，`UITextBorderStyleRoundedRect` とは，`UITextField` の見栄えよいスタイルのことです．

`rake` してインターフェースがどうなっているか確かめましょう :

![search controller in app](images/1.png)

さてそろそろ幾つかのイベントにフックを仕掛けましょうか．私が `BubbleWrap` にはいくつか気の利いたラッパーがあると言ったのを覚えていますか？実のところ，`addTarget:action:forControlEvents` という扱いにくいシンタックスを置き換えるために，既に一度使っていますね．Ruby 風に書くことで，コードがどれだけ明確になるかみてみましょう :

```ruby
  def viewDidLoad
    ...

    self.view.addSubview @search

    @search.when(UIControlEventTouchUpInside) do
      @search.enabled = false
      @text_field.enabled = false

      hex = @text_field.text
      # chop off any leading #s
      hex = hex[1..-1] if hex[0] == "#"

      Color.find(hex) do |color|
        @search.enabled = true
        @text_field.enabled = true
      end
    end
  end
```

`when` 関数は，全ての `UIControl` サブクラス (`UIButton` もその一つです ) で利用可能であり，`UIControlEvent` のビットマスクを引数として取ります．リクエストの実行中，私たちは一時的に UI 要素を無効にします．

あれ……`Color.find` メソッドとは何でしょうか？ええと，URL リクエストはコントローラではなくモデルの中で扱う方がおすすめです．その方が，アプリの他の部分から，サーバーにある `Color` を取得しようとした時に，コードの重複がなくなります．*……この言葉が伏線となっていたことを，このとき私たちは知る由もなかった*．

ともかく，`Color` クラスに静的メソッド `find` を追加しましょう :

```ruby
class Color
  ...

  def self.find(hex, &block)
    BW::HTTP.get("http://www.colr.org/json/color/#{hex}") do |response|
      p response.body.to_str
      # for now, pass nil.
      block.call(nil)
    end
  end
end
```

変に見えますか？基本的な `HTTP.get` を使って，API URL 経由でサーバーからデータを取得します．`&block` 表記を使うことで，この関数がブロックと一緒に使われることを明確に表すようにしています．このブロックは別の引数として明示的に渡されるわけではなく，メソッドの後に `do/end` を書くことで暗黙的に渡されます．`.call(some, variables)` の変数の数と順序は `do |some, variables|` に対応しています．

とにかく，`rake` して "3B5998" のような色を渡してみましょう．コンソールでは，このような出力が表示されるはずです :

```
(main)> "{\"colors\": [{\"timestamp\": 1285886579, \"hex\": \"ff00ff\", \"id\": 3976, \"tags\": [{\"timestamp\": 1108110851, \"id\": 2583, \"name\": \"fuchsia\"}, {\"timestamp\": 1108110864, \"id\": 3810, \"name\": \"magenta\"}, {\"timestamp\": 1108110870, \"id\": 4166, \"name\": \"magic\"}, {\"timestamp\": 1108110851, \"id\": 2626, \"name\": \"pink\"}, {\"timestamp\": 1240447803, \"id\": 24479, \"name\": \"rgba8b24ff00ff\"}, {\"timestamp\": 1108110864, \"id\": 3810, \"name\": \"magenta\"}]}], \"schemes\": [], \"schemes_history\": {}, \"success\": true, \"colors_history\": {\"ff00ff\": [{\"d_count\": 0, \"id\": \"4166\", \"a_count\": 1, \"name\": \"magic\"}, {\"d_count\": 0, \"id\": \"2626\", \"a_count\": 1, \"name\": \"pink\"}, {\"d_count\": 0, \"id\": \"24479\", \"a_count\": 1, \"name\": \"rgba8b24ff00ff\"}, {\"d_count\": 0, \"id\": \"3810\", \"a_count\": 1, \"name\": \"magenta\"}]}, \"messages\": [], \"new_color\": \"ff00ff\"}\n"
```

こいつは……すごく…… JSON 風です．そうじゃないですか？( もしあなたが URL に /json/ が含まれていることに気付かなかったなら，JSON 「風」であると思ってくれるはず )．解析して普通の Ruby ハッシュにできたなら素晴しいですよね？

BubbleWrap が再び助けてくれます！私たちの気の利く友人は `BW::JSON.parse` という，名は体を表すようなメソッドを持っています．それを使って `Color.find` を更新しましょう :

```ruby
def self.find(hex, &block)
  BW::HTTP.get("http://www.colr.org/json/color/#{hex}") do |response|
    result_data = BW::JSON.parse(response.body.to_str)
    color_data = result_data["colors"][0]

    # Colr will return a color with id == -1 if no color was found
    color = Color.new(color_data)
    if color.id.to_i == -1
      block.call(nil)
    else
      block.call(color)
    end
  end
end
```

そして `SearchController` の中で，無効な色を取得した場合にコールバックが適切な扱いをできるようにします :

```ruby
  def viewDidLoad
    ...

      Color.find(hex) do |color|
        if color.nil?
          @search.setTitle("None :(", forState: UIControlStateNormal)
        else
          @search.setTitle("Search", forState: UIControlStateNormal)
          self.open_color(color)
        end

        @search.enabled = true
        @text_field.enabled = true
      end
    end
  end

  def open_color(color)
    p "Opening #{color}"
  end
```

よさそうですね．JSON をパースして，存在しない/-1 の id を確認して，それに応じて UI を変更します :

![search controller in app](images/2.png)

素晴らしい！`open_color` メソッドが動くように修正してみましょう．それは `ColorController` へ適切な色と共に push するべきで，今はこんな実装で埋めておきましょうか．

```ruby
def open_color(color)
  self.navigationController.pushViewController(ColorController.alloc.initWithColor(color), animated:true)
end
```

## ColorController

`ColorController` のためにカスタムしたイニシャライザを使うようにしましょう．これらのカスタムイニシャライザは，常に最初にスーパークラスのイニシャライザ ( この場合は `initWithNibName:bundle:`) を呼ぶように設計されています．コントローラのビューには，二つの部分があります : 色のタグを表示するための `UITableView`，そして色の表示と新しいタグを追加する部分．新しいタグを追加したい場合，POSTリクエストが送信され，それに応じてデータが更新されます．

たくさんのことをしているように見えますね．一つずつ進めていきましょう．最初に，カスタムしたイニシャライザについてです :

```ruby
class ColorController < UIViewController
  attr_accessor :color

  def initWithColor(color)
    initWithNibName(nil, bundle:nil)
    self.color = color
    self
  end

  ...
```

iOS の SDK のイニシャライザをオーバーライドする場合は，次の 2 つのことを行う必要があります : あらかじめデザインされたイニシャライザを呼び出し，最後に `self` を返す．いつもの Ruby の `initialize` 関数は使用できません．ご注意あれ！

次にインターフェースを並べてみましょう :

```ruby
  def viewDidLoad
    super

    self.title = self.color.hex

    # A light grey background to separate the Tag table from the Color info
    @info_container = UIView.alloc.initWithFrame [[0, 0], [self.view.frame.size.width, 110]]
    @info_container.backgroundColor = UIColor.lightGrayColor
    self.view.addSubview @info_container

    # A visual preview of the actual color
    @color_view = UIView.alloc.initWithFrame [[10, 10], [90, 90]]
    # String#to_color is another handy BubbbleWrap addition!
    @color_view.backgroundColor = String.new(self.color.hex).to_color
    self.view.addSubview @color_view

    # Displays the hex code of our color
    @color_label = UILabel.alloc.initWithFrame [[110, 30], [0, 0]]
    @color_label.text = self.color.hex
    @color_label.sizeToFit
    self.view.addSubview @color_label

    # Where we enter the new tag
    @text_field = UITextField.alloc.initWithFrame [[110, 60], [100, 26]]
    @text_field.placeholder = "tag"
    @text_field.textAlignment = UITextAlignmentCenter
    @text_field.autocapitalizationType = UITextAutocapitalizationTypeNone
    @text_field.borderStyle = UITextBorderStyleRoundedRect
    self.view.addSubview @text_field

    # Tapping this adds the tag.
    @add = UIButton.buttonWithType(UIButtonTypeRoundedRect)
    @add.setTitle("Add", forState:UIControlStateNormal)
    @add.setTitle("Adding...", forState:UIControlStateDisabled)
    @add.setTitleColor(UIColor.lightGrayColor, forState:UIControlStateDisabled)
    @add.sizeToFit
    @add.frame = [[@text_field.frame.origin.x + @text_field.frame.size.width + 10, @text_field.frame.origin.y],
                      @add.frame.size]
    self.view.addSubview(@add)

    # The table for our color's tags.
    table_frame = [[0, @info_container.frame.size.height],
                  [self.view.bounds.size.width, self.view.bounds.size.height - @info_container.frame.size.height - self.navigationController.navigationBar.frame.size.height]]
    @table_view = UITableView.alloc.initWithFrame(table_frame, style:UITableViewStylePlain)
    self.view.addSubview(@table_view)
  end
```

＿人人人人人人人人＿  
＞　大量のコード　＜  
￣Y^Y^Y^Y^Y^Y^Y￣  

でも，恐れないでください．よくみると，全てここまでにやってきたことで構築しています．ただサブビューをまとめたものを追加して，適切なデータに繋いだだけです．

`rake` して確かめてみましょう :

![color controller in app](images/3.png)

そういえば，デザインでは賞を期待できないこと，すでにお伝えしてましたよね．

タグを繋げていきましょう．タグをテーブルに格納するのに便利なテーブルビューの `delegate` メソッドを使うつもりです．それだけで，風変りな部分やコールバックのない，普通のリストになります :

```ruby
  def viewDidLoad
    ...

    @table_view.dataSource = self
  end

  def tableView(tableView, numberOfRowsInSection:section)
    self.color.tags.count
  end

  def tableView(tableView, cellForRowAtIndexPath:indexPath)
    @reuseIdentifier ||= "CELL_IDENTIFIER"

    cell = tableView.dequeueReusableCellWithIdentifier(@reuseIdentifier) || begin
      UITableViewCell.alloc.initWithStyle(UITableViewCellStyleDefault, reuseIdentifier:@reuseIdentifier)
    end

    cell.textLabel.text = self.color.tags[indexPath.row].name

    cell
  end
```

また `rake` してみると……おおっ！目を引くデータがそこに！

![color tags in app](images/4.png)

もう一つやりましょう : 新しいタグを追加できるようにします．その新機能を実装するには 2 つの方法があります，1 つは `Tag.create(tag)` のようにするか `color.tags << tag` をハックする方法，しかし今回はもう 1 つの `color.add_tag(tag, &block)` の方法でやっていきます．なぜでしょう？なぜなら，その方が色とタグが密結合であることをよく示しているからです．

`add_tag` メソッドは次のようになります :

```ruby
  def add_tag(tag, &block)
    BW::HTTP.post("http://www.colr.org/js/color/#{self.hex}/addtag/", payload: {tags: tag}) do |response|
      block.call
    end
  end
```

ここでは `block.call` へ何も引数を渡していません．成功か失敗といった情報を渡すこともできますが，この例では別にフォールトトレラントでなくともかまわないので，このようにしました．

さて，`ColorController` にあるボタンのコールバックの中に `add_tag` を書きます．タグがサーバーに送信された後，サーバーがタグを本当に受信したか私たちが確認するために，現在のサーバーのデータで手元の色 ( とタグ ) をリフレッシュしたいです．それでは，その処理を `when` コールバックの中に書いていきましょう :

```ruby
  def viewDidLoad
    ...

    self.view.addSubview(@add)

    @add.when(UIControlEventTouchUpInside) do
      @add.enabled = false
      @text_field.enabled = false
      self.color.add_tag(@text_field.text) do
        refresh
      end
    end

    ...
  end

  def refresh
    Color.find(self.color.hex) do |color|
      self.color = color

      @table_view.reloadData

      @add.enabled = true
      @text_field.enabled = true
    end
  end
```

一連の処理を追ってみましょう．`@add` へ `UIControlEventTouchUpInside` コールバックを追加します．そのコールバックでは `color.add_tag` を呼び出し，適切な POST リクエストを実行します．そのリクエストが完了したあと，`refresh` メソッドを呼び出します．そのメソッドは `Color.find` リクエストを実行し，手元のデータをリセットします．

`rake` して，タグを追加してみます．サクッとできましたか．

![adding a tag](images/5.png)

## まとめ

ふー，大きな例ですね．ちゃんと設計がなされており，モデルとコントローラ間の責任を分離する一つの方法を示しています．ビューで様々なことをしたり，KVO を追加したり，もっと色々なことができたでしょうが，この程度の例では，それはやり過ぎだと思います．

この例から何を学びましょうか？

- データを表現するのに，`JSON.parse` で得られるハッシュを直接使わず，モデルを使います．
- モデルの中で URL リクエストを実行します．
- コントローラを，コー​​ルバックとユーザーイベントに応答するのに使います．
- 時間のかかるリクエストが実行されている間，UIを無効にしたり変更して，インターフェイスの応答性を保ちます．

[colr]: http://www.colr.org/api.html
