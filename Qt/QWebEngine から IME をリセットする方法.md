---
tags:
  - QWebEngine
  - QWebChannel
  - IME
  - QInputMethod
---

# QWebEngine から IME をリセットする方法（QWebChannel利用）


## 概要

Qtでは、IME（Input Method Editor）の変換状態をリセットするために `QInputMethod::reset()` が用意されています。

```cpp
QGuiApplication::inputMethod()->reset();
```

Webコンテンツ（JavaScript）からこのAPIを直接呼び出すことはできないため、**QWebChannel** を介して C++ 側へ要求を渡す構成になります。

---

# 基本構成

```
JavaScript
    │
    │ QWebChannel
    ▼
QObject (Bridge)
    │
    ▼
QGuiApplication::inputMethod()->reset()
    │
    ▼
Platform Input Context
    │
    ▼
IBus / Fcitx などの IME
```

---

# C++側

Bridgeクラスにスロットを用意します。

```cpp
class ImeBridge : public QObject
{
    Q_OBJECT

public slots:
    void resetIme()
    {
        if (auto *im = QGuiApplication::inputMethod()) {
            im->reset();
        }
    }
};
```

QWebChannelへ登録します。

```cpp
auto *channel = new QWebChannel(page);
auto *bridge = new ImeBridge();

channel->registerObject(QStringLiteral("imeBridge"), bridge);
page->setWebChannel(channel);
```

---

# JavaScript側

```javascript
new QWebChannel(qt.webChannelTransport, function(channel) {
    window.imeBridge = channel.objects.imeBridge;
});
```

リセットしたいタイミングで呼び出します。

```javascript
imeBridge.resetIme();
```

---

# reset() と commit() の違い

Qtには似たAPIとして `commit()` もあります。

|API|動作|
|---|---|
|`commit()`|未確定文字列を確定する|
|`reset()`|IMEの変換状態をリセットする（未確定文字列はキャンセルされる場合がある）|

どちらを使用するかは、アプリケーションの仕様によって決定します。

---

# API設計のおすすめ

JavaScriptから直接 `resetIme()` を公開するよりも、**用途を表すAPI** を公開する方が保守しやすくなります。

例：

```cpp
public slots:
    void onInputBlurred();
    void cancelComposition();
```

JavaScript側は

```javascript
bridge.onInputBlurred();
```

のように呼び出します。

これにより、将来

- `commit()`
    
- `reset()`
    
- 仮想キーボードを閉じる
    
- ログ出力
    
- プラットフォーム依存処理
    

などを追加しても、JavaScript側を変更せずに済みます。

---

# QWebEngineで利用する際の注意点

QWebEngineでは

- JavaScriptの `blur`
    
- Chromium内部のIME状態
    
- Qtの `QInputMethod`
    
- 仮想キーボード
    

が完全に同期しているとは限りません。

そのため、JavaScriptからは「IMEをリセットする」という低レベルの命令ではなく、

- 入力欄からフォーカスが外れた
    
- 入力をキャンセルした
    
- 仮想キーボードを閉じたい
    

といった**意味を持つイベント**をQWebChannel経由で通知し、C++側で `commit()` と `reset()` を適切に選択する設計が推奨されます。

---

# まとめ

- `QGuiApplication::inputMethod()->reset()` はIMEの変換状態をリセットするAPI。
    
- Webコンテンツから利用するには QWebChannel を経由して C++ 側で呼び出す。
    
- `commit()` と `reset()` は役割が異なるため、仕様に応じて使い分ける。
    
- JavaScriptには低レベルAPIではなく、「入力完了」「入力キャンセル」など意味を持つAPIを公開する方が保守性が高い。
