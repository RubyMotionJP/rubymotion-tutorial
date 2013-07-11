---
layout: default
title: Controllers
description: Use UIViewControllers to organize your app
categories:
- chapters
---

# Controllers

ビューについていくつかふれてきましたが、しかしそれは iOS SDK が使用する "Model-View-Controller" パラダイムの 1 つです。MVC は複雑に聞こえますが、実際には非常にシンプルです。

それはコードの中に、ビュー (みなさんは既にそれを使いました)、モデル (データを表現し、処理します)、そしてコントローラの 3 種類のクラスを持っているべきという考えです。

それでは、コントローラとは何でしょうか？コントローラはモデルとビューの間の層として動作するオブジェクトで、ユーザからのイベントに応答しモデルの変更やビューの更新を行います。とても良く書かれたコードでは、ボタンをタップしたときに、コントローラはイベントを奪い、データのプロパティを更新し、新しいデータを反映するためにビューを変更します。

「全体像」の類いのように聞こえますが、コントローラにはいくつか本当に実用的なことがあります。

- **ビューの再利用**. 投稿に関する情報 `Post` (投稿内容、投稿者、"Likes" など) をすべて表示する `PostView` を考えてみましょう。メインのフィードやユーザプロフィールフィードなど、異なる画面でこのビューを使用したいです。ビューを再利用できるように維持するため、`PostView` は情報を取得する方法を扱うべきではありません。その代わり、コントローラでそれを処理し、処理されたデータをビューへ渡すべきです。
- **プレゼンテーションの管理**. 画面全体を取り扱うビューが欲しいときもあれば、同じことをモーダルボックスに表示するものが欲しい場合があります (iPad と iPhone を考えると)。プレゼンテーションのスタイルだけが異なる、まったく同じ 2 つのビュークラスを記述することは賢明ではないので、コントローラを使いビューにあわせてリサイズやアニメーションを行います。

MVC についてモデルとビューで技術的に難しいことはありません。もしみなさんが MVC を採用すれば、より強固で管理が容易なコードとなることでしょう。

iOS の世界では、コントローラは `UIViewController` です。`UIViewController` には 1 つの `view` プロパティとビューのライフサイクルのようなことを対処するためのメソッドがあり、またデバイスの向きの変更 (横向きから縦向きへ変更など) を対処します。焦らないでください、私たちはすぐにそのライフサイクルに取りかかります。

コントローラが何か、それがすべきことが何かを知りました。では、何を *すべきではない* のでしょう？

- データを直接参照したり、保存すること。コントローラで HTTP リクエストを送信することは魅力的ですが、これらはモデルで行うのがベストです。
- 複雑なビューレイアウトを扱うこと。もし、コントローラの `view` に 1 つ以上のサブビューを直接追加しているのでしたら、ビューでそれを行うようにビューを書き直すべきです。コントローラに書かれている `addSubview` は `self.view.addSubview` だけというのが、良い経験則です。

OK、説明は終わりです。マイケル・ベイ監督がアクションを撮影する時間ですね。

## Everything Is Under Controllers

`./app/controllers` ディレクトリを作成し (`mkdir ./app/controllers`)、`TapController.rb` ファイルをディレクトリ内に追加します。さぁ、次のようにコントローラを書き始めましょう。

```ruby
class TapController < UIViewController
  def viewDidLoad
    super

    self.view.backgroundColor = UIColor.redColor
  end
end
```

`viewDidLoad` は `UIViewController` のライフサイクルを形作るメソッドの 1 つで、`self.view` で示されるビューが作成され、サブビューが追加できるようになると呼び出されます。今日のところは、赤い背景色のビューとしておきましょう。

皆さんは、絶対、問答無用に `viewDidLoad` で `super` を呼び出す必要があります。そうでなければ、良くないことが起こるでしょう。分かりました？

さて、`AppDelegate` へ戻り、以前の `UIView` のコードを削除しましょう。私たちは 1 行追加する必要があり、次のようになります。

```ruby
class AppDelegate
  def application(application, didFinishLaunchingWithOptions:launchOptions)
    @window = UIWindow.alloc.initWithFrame(UIScreen.mainScreen.bounds)
    @window.makeKeyAndVisible

    # This is our new line!
    @window.rootViewController = TapController.alloc.initWithNibName(nil, bundle: nil)

    true
  end
end
```

`rootViewController=` の呼び出しが分かりますか？ window が先ほど定義された `UIViewController` を受け取り、window にフィットするようにその `view` のサイズを調整します。これは、window をセットアップするための良い方法です(`window.addSubview` とは対照的に)。

追加した行でほかに新しい部分は `initWithNibName:bundle:` です。一般的に、これは `NIB` ファイルからコントローラを読み込むために使われます。`NIB` は Xcode の Interface Builder を使うことで作られます。私たちのコントローラのためには Interface Builder を使っていないので、2 つの引数の両方に nil を渡すことができます。

`initWithNibName:bundle:` は `UIViewController` のイニシャライザでもあります。コントローラを作成するたびに、このメソッドを呼び出します。

エディタを邪魔にならないところにどけておいて、`rake` を実行してチェックしてみましょう。次のように表示されます。

![first controller](images/1.png)

大事は小事より起こります。さぁ、コントローラに小さな変更を行いましょう。

```ruby
  def viewDidLoad
    super

    self.view.backgroundColor = UIColor.whiteColor

    @label = UILabel.alloc.initWithFrame(CGRectZero)
    @label.text = "Taps"
    @label.sizeToFit
    @label.center = CGPointMake(self.view.frame.size.width / 2, self.view.frame.size.height / 2)
    self.view.addSubview @label
  end
```

ワイルドな `UILabel` が登場です！`UILabel` は `text` プロパティで静的なテキストを表示するためのビューです。テキストに "Taps" を設定し、サブビューとしてそれを追加しています。

画面に表示するテキストの正確なサイズがまだ分からないので、`UILabel` を初期化するときに、`CGRectZero` を frame に使います。しかし、`sizeToFit` を呼び出すと、内容に合わせて完全にフィットするようにラベルがリサイズします。そして便利な `center` プロパティを使い、ラベルをコントローラのビューの中央に揃えます。

楽しいアプリのために、再び `rake` を実行します。今のところアプリは・・・何もしませんが、それは次の章で。

![with a label](images/2.png)

## Wrap Up

今回、学んだことは何だった？

- iOS SDK は Model-View-Controller を使います。
- `UIViewController` はコントローラの一部を作成し、カスタマイズのために `UIViewController` のサブクラスを作ります。
- コントローラをセットアップするときには `viewDidLoad` を使い、*super* を呼び出しておくことを忘れてはいけません。
- `UIWindow` はコントローラを表示するために `rootViewController` プロパティを持ってます。

[次の章へ棒高跳び。Container Controllers で遊ぼう！](/4-containers)
