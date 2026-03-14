---
title: "Cmd+Shift+3で全画面キャプチャされるのがストレスだったので、ディスプレイ単位でキャプチャするアプリを作った"
emoji: "📸"
type: "tech"
topics: ["macOS", "Swift", "ScreenCaptureKit", "OSS", "ClaudeCode"]
published: true
---

合同会社ハットウの taniura です。

## 3枚目のモニタだけ撮りたいだけなのに

マルチディスプレイで開発していると、こういう場面があります。

**Claude Codeに画面を見せたい。**

右のモニタに出ているエラー画面をサッと貼りたい。それだけ。

ところがmacOSの `Cmd+Shift+3` は**全ディスプレイを一括キャプチャ**します。3枚モニタがあれば3枚分の画像が保存される。欲しいのは1枚だけなのに。

`Cmd+Shift+4` なら範囲選択できますが、毎回マウスで囲むのは面倒です。同じ画面を何度もキャプチャしたいのに、毎回ドラッグしている。

**ショートカット1つで、指定したディスプレイだけクリップボードにコピーしてくれるツールが欲しい。**

探したけどなかった。じゃあ作る。

## OneScreenSnap

https://github.com/taniurakengo1/OneScreenSnap

macOSのメニューバーに常駐するアプリです。やることはシンプル。

1. ディスプレイごとにショートカットを割り当てる
2. ショートカットを押す
3. そのディスプレイだけがクリップボードにコピーされる
4. `Cmd+V` で貼り付け

以上。ファイルは保存しません。クリップボード直行です。

Claude CodeやChatGPTに画面を貼るのが主な用途なので、**余計なステップを全部省いた**設計にしています。

## 機能

### ディスプレイ全体キャプチャ

各ディスプレイにショートカットを割り当てて、ワンキーでキャプチャ。設定画面ではディスプレイの物理配置を図で表示するので、3枚のモニタが全部同じ解像度でも「どれが左でどれが右か」が一目でわかります。

### リージョンプリセット

「このディスプレイのこの範囲」を保存しておいて、ショートカットで毎回同じ範囲をキャプチャ。

ChatGPTの出力エリアだけ、ターミナルの下半分だけ、みたいな使い方ができます。一度範囲を登録すれば、あとはショートカット一発。

### AI最適化リサイズ

Claude / ChatGPTに画像を渡すとき、4Kモニタのフル解像度は大きすぎます。トークンを余計に消費するし、表示にも時間がかかる。

「AI最適化」モードにすると長辺1568pxに自動リサイズしてからクリップボードにコピーします。Anthropicの推奨サイズです。

### キャプチャ通知

キャプチャ成功時にシャッター音 + 画面フラッシュで通知。「フラッシュのみ」「なし」にも変更可能。

## 技術的な話

### ScreenCaptureKit

macOS 14+の`SCScreenshotManager`を使っています。`CGDisplayCreateImage`でもスクリーンキャプチャはできますが、ScreenCaptureKitのほうが権限管理がきちんとしていて、将来的にも安心。

```swift
let filter = SCContentFilter(display: display, excludingWindows: [])
let config = SCStreamConfiguration()
config.width = Int(display.width) * scaleFactor
config.height = Int(display.height) * scaleFactor
config.showsCursor = false

let image = try await SCScreenshotManager.captureImage(
    contentFilter: filter,
    configuration: config
)
```

### ディスプレイの識別

マルチディスプレイ環境で困るのが、ディスプレイIDが再接続で変わること。USB-Cを抜き差しするたびにIDが変わるので、IDベースでショートカットを保存すると設定が壊れます。

OneScreenSnapでは「ディスプレイ名 + 解像度」をstable keyとして使っています。

```swift
public var stableKey: String {
    "\(name)_\(Int(bounds.width))x\(Int(bounds.height))"
}
```

同じモデルで同じ解像度のモニタが複数ある場合は区別できませんが、設定画面に物理配置図と位置ラベル（← 左 / 中央 / 右 →）を表示することで対処しています。

### グローバルショートカット

`CGEventTap`でキーボードイベントを監視しています。アクセシビリティ権限が必要ですが、macOS標準のスクリーンキャプチャと同じ操作感を実現するにはこれが一番確実。

### ショートカット重複検出

ディスプレイ用とリージョン用のショートカットは共通のプールで管理していて、重複を検出したら確認ダイアログを出します。「置き換え」を選ぶと古い方が自動で解除。

## インストール

```bash
git clone https://github.com/taniurakengo1/OneScreenSnap.git
cd OneScreenSnap
sudo make install
make start
```

macOS 14 (Sonoma) 以降、Xcode Command Line Toolsが必要。

初回キャプチャ時に「画面収録」の権限を求められるので許可してください。

## こんな人向け

- マルチディスプレイで開発している
- Claude Code / ChatGPT / Copilotに画面を頻繁に貼る
- `Cmd+Shift+3` の全画面キャプチャにイラっとしたことがある
- 毎回 `Cmd+Shift+4` で範囲選択するのが面倒

## まとめ

macOSのスクリーンキャプチャ、マルチディスプレイだと地味に不便なんですよね。

「このディスプレイだけ撮りたい」という単純な要望なのに、標準機能では意外とできない。なので作りました。

OSSとして公開しています。同じ悩みを持つ人の役に立てば幸いです。

https://github.com/taniurakengo1/OneScreenSnap
