---
layout: default
title: Animations
description: Animate UIViews in RubyMotion with CoreAnimation
categories:
- chapters
---

# Animations

みなさんが iOS と他のモバイル OS との最大の違いについてユーザに尋ねると、アニメーションは間違いなくトップの答えのひとつとなるでしょう。フェード、フリップ、回転、バウンド、カールは Apple のアプリにありふれています。

これらのいくつかはデフォルトの UI 要素 (`UINavigationController`, `UITableView` など) で顕著です。そして本当に素晴らしいことは、それらはすべて同じ API に基づいて構築されているということです。文字通り、`UIView` はたった 1 行のコードでアニメーションできます。それは信じられないくらい素晴らしいことです。

さぁ、さっそくいくつか遊んでみましょう。画面の周りにボックスを移動させてみましょう。私を信じてください。コードは今述べた通りにシンプルです。

`window` にビューを追加することから始め、そして設定したポイントの周辺でアニメーションします。これは比較的に短いので、ここで `AppDelegate` のすべての変更を破棄しましょう。

```ruby
def application(application, didFinishLaunchingWithOptions:launchOptions)
  @window = UIWindow.alloc.initWithFrame(UIScreen.mainScreen.applicationFrame)
  @window.makeKeyAndVisible

  # the points we're going to animate to
  @points = [[0, 0], [50, 0], [0, 50], [50, 50]]
  @current_index = 0

  # usual method of adding views to our window
  @view = UIView.alloc.initWithFrame [@points[@current_index], [100, 100]]
  @view.backgroundColor = UIColor.blueColor
  @window.addSubview(@view)

  animate_to_next_point

  true
end

def animate_to_next_point
  @current_index += 1

  # keep current_index in the range [0,3]
  @current_index = @current_index % @points.count

  UIView.animateWithDuration(2,
    animations: lambda {
       @view.frame = [@points[@current_index], [100, 100]]
    },
    completion:lambda {|finished|
      self.animate_to_next_point
    }
  )
end
```

このコードには新しく登場したものが 2 つあります。最初に、`[[0, 0], [100, 100]]` という構文は `frame` のためのものです。入れ子にした配列はこれまで使ってきた `CGRectMake` を簡略化したものです。最初の配列は `origin` の値を表し、二つ目の配列は `size` の値を表しています。`frame` を変更するために `@points` を簡単に使用することができ、素晴らしいことです。

二つ目は `lambda` です。この用語に慣れていない人向けに説明すると、`lambda` は変数が割り当てられたり引数が渡されたりする関数です。たとえば、`my_lambda = lambda { p "Hello!" }` とすることができます。ここで `my_lambda.call` を実行すると、`lambda` ブロック内のコードが実行されます。`lambda` は引数を持つこともできます。

```ruby
my_lambda = lambda { |name| p "Hello #{name}!" }
my_lambda.call("Clay")
=> Hello Clay
```

さて、アニメーションの例に戻りましょう。いつものような window を作成し、ボックスを移動するためにいくつかポイントを定義します。window にボックス (`@view`) を追加し、`animate_to_next_point` を開始することでよく知られたマジックが起こります。

`UIView.animateWithDuration:animations:completion:` はアニメーションを制御するものです。たくさんのパーツがこのメソッドにあるので、ひとつずつ見ていきましょう。

`duration` にはアニメーションをどのくらい継続するかを秒単位で設定します。`1.85` のように float の値を使うこともできます。大まかには "素早い" アニメーションは `.3` 秒にするとよいです。

*任意の* ビューに対して `animations:` の lambda で行った変更がスムーズにアニメーションされます。なんてクールなんでしょう！？Android と比較すると、Android では望むアニメーションのために各プロパティの独自の `Animation` オブジェクトを作成する必要があります。さて、いくつかのプロパティはアニメーションできませんが、`frame`, `bounds` や `alpha` といった一般的なものは問題なく動きます (完全なリストを見たければ、 [こちら][1] をチェックしてください)。私たちの場合では、`@view.frame` の値を `@points` で用意してある次の origin に設定します。

最後に、`completions:` はアニメーションが終了した後で呼び出されます。この lambda では *必ず* 引数を受け付ける必要があります。アニメーションが本当に終了したかを通知する `boolean` の値になります。他のアニメーションを明示的に行う場合にのみ、それによってアニメーションが取り消されるため、アニメーションは失敗となることがあります。

`rake` を実行し、シミュレータの左上の角をスライドする青いボックスが表示されます。とても簡単でしたよね？

![simple box animations](images/1.png)

私はこんなに簡単であったという衝撃を忘れることができません。さきほどのアニメーションに 1) delay を持たせる 2) 別の "アニメーションカーブ" を持たせる、と変更することでスパイスを振りかけましょう。慣れていない？現在のアニメーションを見てみましょう。各ポイントの間で、ボックスがどのくらいスピードアップしスローダウンしているかわかりますか？もっと近くで見てください。必ず分かるはずです。

この動きになる理由は、iOS のデフォルトのアニメーションカーブが `UIViewAnimationOptionCurveEaseInOut` だからです。実際の生活のなかで頻繁におこる動き方をシミュレートしていて本当に素晴らしいエフェクトです。私たちは素早く方向転換しませんし、徐々にスピードをあげます。いろいろ遊ぶために、別のカーブを使ってみましょう。

これまで使ってきた `animateWithDuration:animations:completion:` をより設定項目のあるものに変更します。`animateWithDuration:delay:options:animations:completion:` はアニメーションを開始する前に delay を追加し、また `options` を設定できます。`delay` は秒単位の値を取り、`animations:` ブロックの開始までのオフセットとなります。

`options` は少し複雑です。`UIViewAnimationOption` というプレフィックスの値で integer のビットマスクを取ります。`(0 | 1 | 4) == 5` のように Ruby でビットマスクを行うことができます。リニアなアニメーションカーブと自動的にアニメーションを繰り返すというような、同時に複数のアニメーションオプションを組み合わせたい場合に備えビットマスクになっています(`options: (UIViewAnimationOptionCurveLinear | UIViewAnimationOptionRepeat)`)。

(`UIView.animate...` でより設定項目が少ない `animateWithDuration:animations:` というものもあります)。

とにかく、動かしてみましょう。`animate_to_next_point` メソッドを次のように変更します。

```ruby
def animate_to_next_point
  @current_index += 1
  @current_index = @current_index % @points.count

  UIView.animateWithDuration(2,
    delay: 1,
    options: UIViewAnimationOptionCurveLinear,
    animations: lambda {
       @view.frame = [@points[@current_index], [100, 100]]
    }, completion:lambda {|finished|
      self.animate_to_next_point
    })
end
```

`rake` を実行し、新しいアニメーションをチェックしましょう。

古くさいもののように見えますが、その動作はかなり異なっていますよね？ポイント間でいくらか時間がかかるようになり、スライドはよりメカニカルです。

## You Know This Section

iOS の基本的なアニメーションを扱ってきました。これまでどんなことをやってきましたか？


- `animateWithDuration:animations:` と類似したものを使い、ビューのプロパティを `animations:` の lambda で変更することでアニメーションが動きました。
- アニメーションカーブによってアニメーションの動作が決まります。`UIViewAnimationOptionCurveLinear` と `UIViewAnimationOptionCurveEaseInOut` を例として扱いました。

[次の章ではモデルを学びます。いつやるか？今でしょう！](/7-models)

[1]: http://developer.apple.com/library/ios/#documentation/uikit/reference/uiview_class/uiview/uiview.html#Overview_section
