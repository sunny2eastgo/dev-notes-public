---
tags:
  - touch-action
  - QWebEngine
  - Chromium
  - ジェスチャ
---

## はじめに

ブラウザ標準のスクロールやピンチズームを抑止する際の touch-action の挙動を整理する。
ジェスチャは、ブラウザやOSで処理や挙動が変わってくるため、Chromiumベースのブラウザを前提に進める。

## touch-action の基本

`touch-action` は 設定された要素がブラウザに対して
どのタッチジェスチャを許可するかを宣言する CSS プロパティである。

代表的な値は次の通り。

| 値            | 意味                          |
| ------------ | --------------------------- |
| auto         | ブラウザにすべて任せる                 |
| none         | パン・ピンチズームなどを禁止              |
| pan-x        | 横スクロールのみ許可                  |
| pan-y        | 縦スクロールのみ許可                  |
| pan-x pan-y  | スクロールのみ許可                   |
| pinch-zoom   | ピンチズームのみ許可                  |
| manipulation | パン・ピンチズームを許可するがダブルタップズームは禁止 |

### touch-action の適用範囲

`touch-action` は **同一 Document 内で祖先要素へさかのぼって評価**される。

ブラウザはタッチ開始時に、イベントターゲットから祖先要素へ向かって `touch-action` を評価し、そのジェスチャをブラウザが処理するか、Webアプリケーションへ委ねるかを決定する。

`touch-action`はジェスチャ開始時に一度だけ評価される。ジェスチャ開始後に `touch-action` を変更しても、そのジェスチャには反映されない。

```
Top Document
└── body
    └── div
        └── button
```

例えば

```css
body {
    touch-action: none;
}
```

を指定すると、button にも適用される。

## iframe と Document の関係

```
Top Document(A)
└── html
    └── body (touch-action: none)
        └── iframe 要素
        └── button (パン・ズームなどを禁止)

Iframe Document(B)
└── html
    └── body
	    └── button (パン・ズーム可)
```

Top Document(A)にiframe要素がありiframe要素がDocument(B)をロードする場合、
(A)と(B)は別Documentの扱いになるため
(A)のhtml もしくはbodyで設定した`touch-action`は(B)内の要素には適用されない。

## iframe を動的に生成・破棄する場合の注意点

iframeのDocumentに対して常にブラウザジェスチャを抑止する目的で`touch-action: none` を使用する際は
注意が必要である。

iframe に `touch-action: none` を設定していても、Top Document に設定していない状態になっていると
iframe が表示されていない領域はブラウザジェスチャを受け付ける状態になる。

iframe が存在していても、iframe の外側をタッチした場合は Top Document の `touch-action` が評価される。

また、iframe自体が破棄されたり、再生成するタイミングではブラウザ上には`touch-action: none`が設定されていないTop Documentしかないため、ブラウザジェスチャを受け付ける状態になってしまう。

例えば、

```
Top Document
touch-action: auto

└── iframe
    touch-action: none
```

では、通常は iframe がタッチイベントの対象となるためブラウザジェスチャは抑止される。

しかし、iframe を削除した直後や再生成中は

```
Top Document
touch-action: auto

（iframe なし）
```

となる。

この瞬間は Top Document がタッチ対象となるため、
ブラウザのスクロールやピンチズームが開始される可能性がある。

そのため、ブラウザジェスチャを常に禁止したい場合は、
Top Document にも `touch-action: none` を設定しておくことを推奨する。

## pan と pinch-zoom の違い

- pan
  1本指のドラッグ操作によるスクロール
```
   ↑
← □ →
   ↓

1本指で移動
```
  
- pinch-zoom
  2本指操作による拡大・縮小
```
↖ ↗  
□  
↙ ↘  
  
2本指で拡大・縮小
```

## ダブルタップズームとの関係

`touch-action` の仕様では主に

- pan
- pinch-zoom

が定義されている。

一方、ダブルタップズームはブラウザ固有ジェスチャの要素があり、
Pointer Events の仕様では `pinch-zoom` と同列には扱われていない。

しかし Chromium 系ブラウザでは

```css
touch-action: none;
```

を指定すると、

- パン
- ピンチズーム
- ダブルタップズーム

も抑止されることが一般的である。

なお、これは Chromium の実装によるものであり、
ブラウザ実装によって挙動が異なる可能性がある。

## touch-action と meta viewport の違い

| touch-action        | meta viewport      |
| ------------------- | ------------------ |
| CSS プロパティ           | HTML の meta タグ     |
| ジェスチャを制御            | ページズームを制御          |
| 要素単位に設定可能           | ページ単位              |
| iframe ごとに設定可能      | Top ページ全体へ影響       |
| ジェスチャー開始時まで動的に設定変更可 | ページ初期化時に設定が確定し変更不可 |
`touch-action` はブラウザへ

> この要素ではどのタッチジェスチャを許可するか

を通知する。

一方、

```html
<meta
    name="viewport"
    content="maximum-scale=1,user-scalable=no">
```

は

> ページ全体のユーザーズームを許可するか

を指定する。

`meta viewport` はページ全体に対して有効であり、iframe ごとに異なる設定を適用することは基本的にできない。

目的が異なるため、
まずは `touch-action` による制御を行い、
それでもブラウザズームが抑止できない場合に
`meta viewport` の導入を検討するのがよい。
## Chromium ベースブラウザでの設計例

ブラウザ標準ジェスチャを利用しない場合の設計例を示す。
```
Top
touch-action:none

├── iframe A
│      touch-action:none
│
└── iframe B
       touch-action:auto
```

ブラウザ全体では標準ジェスチャを利用しないが
iframe単位で必要な領域のみジェスチャを有効にしている。(iframe B)

Top Document にも `touch-action: none` を設定することで、
iframe の生成・破棄中もブラウザジェスチャが開始されることを防げる。

## まとめ

- `touch-action` は同一 Document 内でのみ評価される。
- iframe は別 Document のため、`touch-action` は伝播しない。
- Top Document にも `touch-action: none` を設定することで、iframe の再生成中もブラウザジェスチャを抑止できる。
- `pan` と `pinch-zoom` は異なるジェスチャである。
- Chromium 系では `touch-action: none` によりダブルタップズームも抑止されることが多い。
- `touch-action` はジェスチャ制御、`meta viewport` はページズーム制御であり役割が異なる。
- まずは `touch-action` による制御を行い、必要な場合のみ `meta viewport` を導入する。

ブラウザ標準ジェスチャを利用しないアプリケーションでは、  
Top Document を起点として touch-action のポリシーを設計すると、  
iframe の追加・削除があっても一貫した動作を実現できる。