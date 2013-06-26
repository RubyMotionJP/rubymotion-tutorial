---
layout: default
title: Hello Motion
description: Learn the installation and project setup of RubyMotion, then make your first iOS app
categories:
- chapters
---
# Hello Motion

## Installation

[RubyMotion][rm] は [MacRuby][macruby] の開発を指揮してきた方が設立した HipByte 社の商用製品です。[ライセンス購入][buy] の際には、ライセンスキーとインストーラアプリケーションを受け取ります。Mac App Store から、Xcode を入手することも必要となります。Xcode は RubyMotion で必要となるいくつかの開発ツール (たとえば iOS シミュレータなど) をインストールします。実際には、Xcode IDE で RubyMotion プロジェクトを作成することはありません。

その代わりに、RubyMotion ではコマンドラインツールを利用し、みなさんが選択したテキストエディタの利用が推奨されています。多くの有名なエディタには[アドオン][packages]があり、コード補完やビルドを手助けしてくれます。また、RubyMotion は RubyGems や Rake といった既存の Ruby ツール上に環境が構築されます。みなさんが Rubyist でしたら、居心地が良い環境でしょう。

インストールが完了すれば、RubyMotion の世界へダイブできます。

## The First App

Terminal を起動し、RubyMotion プロジェクトを作成する場所へ移動します。次のコマンドを実行します。

```ruby
motion create HelloMotion
```

`motion` は RubyMotion で使用する 2 つのコマンドの 1 つです。もしみなさんが Rails になじみがあれば、`motion` は `rails` コマンドに相当します。そのコマンドはプロジェクトや、RubyMotion ツール自体の管理を行います (ときどき、`motion update` を実行することを思い出させてくれます) 。

`motion create` によって、`HelloMotion` フォルダと、いくつかのファイルがフォルダ内に作られます。`cd` でフォルダ内へ進みましょう (`cd ./HelloMotion`)。このフォルダ内で引き続きコマンドを実行することになるので、Terminal のウィンドウやタブをキープしておきましょう。

作成される 2 つのファイル `Rakefile` と `./app/app_delegate.rb` について説明します。

`Rakefile` はみなさんのアプリケーションの設定 (アプリケーションの名前やどのようなリソースファイルが含まれるかなど) と、ライブラリの導入 (サードパーティの gem やそのほかのローカルのソースファイル) を扱うファイルです。`Rakefile` は RubyMotion ワークフローの残りの 1 つ `rake` コマンドで使用されます。RubyMotion 1.11 において、`Rakefile` は次のように生成されます。

```ruby
$:.unshift("/Library/RubyMotion/lib")
require 'motion/project'

Motion::Project::App.setup do |app|
  # Use `rake config' to see complete project settings.
  app.name = 'HelloMotion'
end
```

もしみなさんが Ruby にあまりなじみが無いのでしたら、「`$:.unshift` とは何者だろうか？」と戸惑うかもしれません。「`require` を使用するときに、必要に応じて '/Library/RubyMotion/lib' からもライブラリを探すように」とこの行で Ruby に指示しています。'motion/project' はそのディレクトリに存在し、`$:.unshift` がなければ見つけることができません。

`require 'motion/project'` は RubyMotion で適切にアクセスが行われ、`.setup` ブロックでアプリケーションの設定が行えるようになります。`rake config` によってリストアップされる項目として、`app` で設定できるすべてのプロパティを確認できます。デフォルトでは、RubyMotion は `.name` にプロジェクト名を設定します。

ところで、なぜ `Rakefile` があるのでしょうか？ RubyMotion はそのすべての機能のために `rake` を利用し、みなさんが  `rake` を実行するディレクトリ内の `Rakefile` が `rake` コマンドによって使用されます。`Rakefile` には rake (`rake <task>`) で実行できる "task" の集合の定義することができ、そして "task" は `require "motion/project"` とした時に実際に用意されます。

とりあえずやってみましょう！ Terminal で `rake` とコマンドを実行すると、空白の iPhone シミュレータがポップアップされることでしょう。さらに、みなさんの Terminal ではインタラクティブコンソールが実行されています。このコンソールの中で、みなさんは新しいコードを実行できます。

![hello](images/0.png)

「やったぜ！」とあなたは歓喜するかもしれませんね。しかし、どのようなことが行われたのでしょうか？ `app.name = 'HelloMotion'` からどのようにして、iPhone のポップアップを得たのでしょうか？

RubyMotion の `App` はいくつかデフォルトのコード、たとえば (最も重要なのは) `./app` 内のすべての Ruby ファイルを再帰的に保有していることがわかります。最初のほうで `app_delegate.rb` について言及したことを覚えているでしょうか？アプリケーションをコンパイルしたときに、そのファイルも含まれていることがわかりますね。さぁ、見てみよう！

```ruby
class AppDelegate
  def application(application, didFinishLaunchingWithOptions:launchOptions)
    true
  end
end
```

う〜ん、1 つのメソッドが `AppDelegate` クラスで定義されているだけです。スーパークラスさえ書かれていません。これは何をどのようにするのでしょうか？

手っ取り早く、`rake config` コマンドを実行してみましょう。いくつかの情報を出力しますが、私たちの関心があることは次の内容です。

```ruby
...
delegate_class         : "AppDelegate"
...
```

なんてこともない。AppDelegate と書かれていますね！ RubyMotion は `delegate_class` に割り当てられた名前のクラスを探し、アプリを起動するときに利用します。`SuperAppDelegate` というクラスが呼ばれるようにする場合には、`Rakefile` を次のものに変更します。

```ruby
Motion::Project::App.setup do |app|
  app.name = 'HelloMotion'
  app.delegate_class = "SuperAppDelegate"
end
```

それで、、、delegate とは何でしょう？ iOS の世界では、ユーザが私たちのアプリケーションを起動したときに、システムはいろいろなものをセットアップします。私たちは処理中にいろいろなイベントに対応できるオブジェクトを OS に与える必要があります。私たちは "application delegate" として、オブジェクトを参照します。アプリケーションの開始・終了、アプリケーションがバックグラウンドになったとき、Push Notification や、すべてのおもしろそうなものを受信したときに、そのオブジェクトはコールバックを受けます。

`motion` コマンドで生成されるコードは、`def application(application, didFinishLaunchingWithOptions: launchOptions)` だけを実装します。それは、通常の Ruby のコードとは少々おもむきが異なっています。なぜなら、`didFinishLaunchingWithOptions` というラベルが途中にあります。それは何でしょう？さて、ちょっとした歴史の授業の時間です。

多くのプログラミング言語では、関数は `obj.makeBox(origin, size)` のような感じで書きます。関数の実装を調べ、何の変数をどこへ入れるか理解する必要があるので、ちょっとした苦痛でしょう。Objective-C はこの問題を対処するために「名前付き引数」を使用します。Objective-C では、同じ関数を `[obj makeBoxWithOrigin: origin andSize: size];` のように書きます。各変数は関数名の一部へ引数として渡されます。変数がどういったことに使われるか明確でしょう？かなり良いですよね。変数の部分にコロンを置き連結することで、 この関数は `makeBoxWithOrigin:andSize:` という名で参照されます。

Ruby では、「名前付き引数」は存在せず、`obj.make_box(origin, size)` という伝統的なメソッドとなります。RubyMotion では
、`obj.makeBox(origin, andSize: size)` のように Apple オリジナルの API と非常に互換性がある「名前付き引数」の実装が追加されています。`andSize` はお世辞でもなく、本当にメソッド名の一部です。`obj.call_method(var1, withParam: var2)` と `obj.call_method(var1, withMagic: var2)` は、`obj.call_method` が 通常の Ruby のような形式にもかかわらず、まったく異なります。

それでは、話を戻しましょう。システムがアプリケーションのセットアップを終了し、私たちによるセットアップを実行する準備が整ったときに、`application:didFinishLaunchingWithOptions:` が呼ばれます。

```ruby
def application(application, didFinishLaunchingWithOptions:launchOptions)
  true
end
```

今のところ、いつも `true` を返すことを前提としています。いくつかのアプリケーションは `launchOptions` をアプリケーションを起動させるかどうかを決定するために使用するかもしれませんが、ほとんど不要でしょう。

さて、アプリケーションが起動する仕組みと、エントリーポイントがわかりました。何かしてみましょうか？ `application:didFinishLaunchingWithOptions:` を次のように変更してみましょう。

```ruby
def application(application, didFinishLaunchingWithOptions:launchOptions)
  alert = UIAlertView.new
  alert.message = "Hello Motion!"
  alert.show

  puts "Hello again!"

  true
end
```

何が増えたでしょう？まず、`UIAlertView` について 3 行あります。`UIAlertView` は、みなさんが iOS を利用しているときに時々見かける (iTunes Store にログインするときや、iOS 5 以前の Push Notification など) 青いポップアップです。私たちはそれを作成し、メッセージを与え、そしてそれを表示します。

次に、`puts` は基本的なログを表示するものです。普通の文字列や、情報が欲しいオブジェクトを表示するために、それに渡すことができます。

再び `rake` を実行しましょう。あぁ、閉じることができない青いポップアップ！そして、Terminal のウィンドウで、コンソール中に "Hello again!" が出力されています。あまり多くの進展はありませんでしたが、少なくとも私たちはここへのたどり着き方がわかりました。

![hello](images/1.png)

まとめ

- `motion create <ProjectName>` で RubyMotion アプリケーションを作成します。
- `Rakefile` 内で、アプリケーションの設定とライブラリの導入を扱います。
- アプリケーションには delegate が必要で、RubyMotion は `Rakefile` で delegate の設定 (あるいはデフォルトを使うか) を行います。
- アプリケーションの delegate では、最初のエントリーポイントとして `application:didFinishLaunchingWithOptions:` が呼ばれます。
- プロジェクトディレクトリ内で、`rake` を使用しアプリケーションを実行します。

[次の章では、画面上にいくつかビューを配置するよ！](/2-views)

[rm]: http://www.rubymotion.com/

[macruby]: http://macruby.org/

[xcode]: http://itunes.apple.com/us/app/xcode/id497799835?mt=12

[buy]: http://sites.fastspring.com/hipbyte/product/rubymotion

[packages]: http://rubymotion.jp/RubyMotionDocumentation/articles/editors/index.html
