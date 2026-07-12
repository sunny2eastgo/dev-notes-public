---
tags:
  - IME
  - QWebEngine
  - Linux
  - Qt5
  - Qt6
  - Chromium
---

## 概要

Qt6 + QWebEngine で Web ページの `<input>` 要素に対して日本語 IME 入力を行う環境において、未確定文字列の入力中にフォーカスアウトした後、再度入力を開始すると、前回の変換状態が残っているような挙動が発生した。

本調査では JavaScript の Composition Event と Qt の IME イベントを比較し、原因を調査した。

---
## 発生条件

Linux + Qt6(Qt5.15でも再現する) + QWebEngine + 日本語IME
# 発生した現象

例として、

```
「あい」
```

まで入力した状態でフォーカスアウトする。

画面上では文字列は正常に確定して見える。

その後、同じ input に再度フォーカスして「あ」を入力すると、

```
あいあいあ
```

となり、本来期待される

```
あいあ
```

にならない。

---

# JavaScriptイベントの確認

フォーカスアウト時のイベントは以下の順序で発生していた。

```
compositionend
blur
focusout
```

`compositionend` の時点では、

```
value = "あい"
```

となっており、DOM上では未確定文字列は確定済みに見える。

---

# Qt側イベントの確認

Qt側では `QInputMethodEvent` を監視した。

しかし、フォーカスアウト時に

```
preedit=""
commit="あい"
```

のような commit イベントは送られてこなかった。

つまり、

- Qt Input Method が commit を実行した形跡はない
    
- IMEコンテキストの終了処理も確認できなかった
    

---

# 問題の原因

今回の現象は、

**Web側の Composition は終了しているが、Qt/IME側の入力コンテキストは終了していない**

ことが原因と考えられる。

概念図で表すと、

```
IME
    │
Qt Input Method
    │
Chromium
    │
DOM(input)
```

フォーカスアウト時には、

```
Chromium
    composition終了
    valueへ反映
    compositionend発生
```

までは行われる。

しかし、

```
Qt Input Method
    preedit状態
    変換コンテキスト
```

はリセットされない。

そのため再度フォーカスすると、

```
以前の preedit
    +
今回の入力
```

として扱われ、

```
「あい」
+
「あ」
↓

「あいあ」
```

という新しい composition が開始されてしまう。

その結果、

```
既存 value
    あい

新しい composition
    あいあ
```

が結合され、

```
あいあいあ
```

となっていた。

---

# 対策

フォーカスアウト時に Qt 側の IME コンテキストを明示的にリセットする。

```
QGuiApplication::inputMethod()->reset();
```

ただし、JavaScript のイベント処理中に即座に呼ぶのではなく、Qt イベントループへ戻してから実行する。

例)

```cpp
QMetaObject::invokeMethod(
    qApp,
    []{
        if (auto *im = QGuiApplication::inputMethod())
            im->reset();
    },
    Qt::QueuedConnection);
```

---

# 対策後の動作

再度ログを取得すると、

```
compositionstart

compositionupdate
data = "あ"

input
value = "あいあ"
```

となった。

以前の

```
data = "あいあ"
```

ではなく、

```
data = "あ"
```

になっていることから、

IME の変換状態は正しくリセットされていることが確認できた。

---

# `commit()` を使用しなかった理由

今回は JavaScript 側で

```
compositionend
```

が発生しており、

DOM の value には既に文字列が反映されている。

一方、Qt 側では commit イベントは発生していない。

この状態で

```cpp
QGuiApplication::inputMethod()->commit();
```

を呼ぶと、

- 二重 commit
    
- 重複入力
    
- プラットフォーム依存の挙動
    

となる可能性がある。

そのため、本件では

```
reset()
```

のみを実行する方針とした。

---

# まとめ

今回の現象は、

- Web の Composition は正常終了している
    
- Qt Input Method の状態だけが残っている
    

ことによって発生していた。

そのため、

```
compositionend
    ↓
blur / focusout
    ↓
QWebChannel
    ↓
QInputMethod::reset()
```

というフローを追加することで、IME の内部状態を明示的に破棄し、再フォーカス時に新しい Composition を開始できるようになった。

Qt6 + QWebEngine では、DOM 上のフォーカス遷移だけでは Qt の入力メソッド状態が自動的にリセットされないケースがあり、このような環境では `QInputMethod::reset()` を明示的に呼び出すことが有効な対策となる。

## コードサンプル

### C++：QWebChannel公開オブジェクト

#### `ImeBridge.h`

```c++
#pragma once

#include <QObject>

class ImeBridge final : public QObject
{
    Q_OBJECT

public:
    explicit ImeBridge(QObject *parent = nullptr);

    Q_INVOKABLE void resetInputMethod();

private:
    bool m_resetScheduled = false;
};
```

#### `ImeBridge.cpp`

```c++
#include "ImeBridge.h"

#include <QDebug>
#include <QGuiApplication>
#include <QInputMethod>
#include <QTimer>

ImeBridge::ImeBridge(QObject *parent)
    : QObject(parent)
{
}

void ImeBridge::resetInputMethod()
{
    /*
     * focusoutが連続して発生した場合に、
     * resetを何度も予約しないようにする。
     */
    if (m_resetScheduled) {
        qDebug() << "[IME] reset already scheduled";
        return;
    }

    m_resetScheduled = true;

    /*
     * QWebChannelの呼び出し処理中にはresetせず、
     * 現在のWeb/Qtイベント処理を抜けてから実行する。
     */
    QTimer::singleShot(0, this, [this]() {
        m_resetScheduled = false;

        QInputMethod *inputMethod = QGuiApplication::inputMethod();

        if (!inputMethod) {
            qWarning() << "[IME] QInputMethod is not available";
            return;
        }

        qDebug() << "[IME] before reset"
                 << "focusObject=" << QGuiApplication::focusObject()
                 << "visible=" << inputMethod->isVisible()
                 << "locale=" << inputMethod->locale();

        inputMethod->reset();

        qDebug() << "[IME] QInputMethod::reset() completed";
    });
}
```

```c++

/* QWebPageを拡張して後からsetPageするケースは
 * webpageのコンストラクタで実行する。
 */
auto *view = new QWebEngineView;

 /*
  * QWebChannelと公開オブジェクトは、
  * ページより長く生存する必要がある。
  */ 
auto *channel = new QWebChannel(view);
auto *imeBridge = new ImeBridge(channel); 

/*
 * JavaScriptからは
 * channel.objects.imeBridge 
 * として参照できる。
 */
channel->registerObject(QStringLiteral("imeBridge"), imeBridge);
view->page()->setWebChannel(channel);
```

### HTML/JavaScript

#### JSファイルの読み込み
```html
<script src="qrc:///qtwebchannel/qwebchannel.js"></script>
<script type="module" src="./ime-controller.js"></script>
```

#### `web-channel.js`

```JavaScript
let channelPromise = null;

export function initializeWebChannel() {
    if (channelPromise) {
        return channelPromise;
    }

    channelPromise = new Promise((resolve, reject) => {
        if (
            typeof qt === "undefined" ||
            !qt.webChannelTransport ||
            typeof QWebChannel === "undefined"
        ) {
            reject(new Error("QWebChannel is not available"));
            return;
        }

        new QWebChannel(qt.webChannelTransport, (channel) => {
            resolve(channel.objects);
        });
    });

    return channelPromise;
}
```

#### `ime-controller.js`

```JavaScript
import { initializeWebChannel } from "./web-channel.js";

let imeBridge = null;
let isComposing = false;
let resetPending = false;

function isImeInputElement(element) {
    return (
        element instanceof HTMLTextAreaElement ||
        (
            element instanceof HTMLInputElement &&
            ["text", "search", "url", "tel", "email", "password"]
                .includes(element.type)
        ) ||
        (
            element instanceof HTMLElement &&
            element.isContentEditable
        )
    );
}

function requestInputMethodReset() {
    if (!imeBridge) {
        return;
    }

    imeBridge.resetInputMethod();
}

document.addEventListener("compositionstart", (event) => {
    if (isImeInputElement(event.target)) {
        isComposing = true;
    }
}, true);

document.addEventListener("compositionend", (event) => {
    if (!isImeInputElement(event.target)) {
        return;
    }

    isComposing = false;

    if (resetPending) {
        resetPending = false;
        requestInputMethodReset();
    }
}, true);

document.addEventListener("focusout", (event) => {
    if (!isImeInputElement(event.target)) {
        return;
    }

    if (isComposing) {
        resetPending = true;
        return;
    }

    requestInputMethodReset();
}, true);

initializeWebChannel()
    .then((objects) => {
        imeBridge = objects.imeBridge;

        if (!imeBridge) {
            throw new Error("imeBridge is not registered");
        }
    })
    .catch((error) => {
        console.error("[IME]", error);
    });
```
