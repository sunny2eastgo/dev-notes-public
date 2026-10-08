# Qt WebEngine：フォーカス中の入力要素をC++で取得する

`document.activeElement` から入力要素の情報を取り出し、`QWebEnginePage::runJavaScript()` のコールバックで受け取る。JavaScriptのオブジェクトを `QVariantMap` に変換し、C++の構造体へ詰め替える。この取得処理だけならQWebChannelは不要。

## 1. JavaScript：情報の抽出とnull対策

次のコード全体を `runJavaScript()` に渡す。トップレベルに直接 `return` を書くと構文エラーになるため、即時実行関数（IIFE）で囲む。

```javascript
(() => {
    // 仮想キーボードで文字を入力する対象かどうかを判定する例。
    function isEditable(el) {
        if (!el) return false;

        // fieldsetによる無効化も含めて除外する。
        if (el.matches(":disabled")) return false;

        if (el.tagName === "TEXTAREA") {
            return !el.readOnly;
        }

        if (el.tagName === "INPUT") {
            const textTypes = [
                "text", "search", "tel", "url",
                "email", "password", "number"
            ];
            return !el.readOnly && textTypes.includes(el.type);
        }

        // 親から継承されたcontenteditableも判定できる。
        return el.isContentEditable === true;
    }

    const el = document.activeElement;

    return {
        editable: isEditable(el),
        type: el?.type || "",
        inputMode: el?.inputMode || "",
        keyboardType: el?.dataset?.keyboardType || "",
        layout: el?.dataset?.layout || ""
    };
})();
```

- `?.` は対象が `null` / `undefined` の場合にアクセスを打ち切る。`el` に加えて `dataset` がない場合も保護する。
- `|| ""` は未定義値や空文字などを空文字にそろえる。この例の対象は文字列なので適している。`0` や `false` を保持したい用途では `??` を使う。
- `isEditable()` は組み込み関数ではなく、用途に合わせて定義する。この例ではボタン・チェックボックス・日付選択などを対象外にする。
- `type` は入力要素なら `text`、`number` など。`textarea` では `textarea` になる。`inputMode` は入力方法のヒントで、入力値の検証規則ではない。

## 2. dataset：HTML属性とcamelCaseの対応

```html
<input type="text"
       inputmode="numeric"
       data-keyboard-type="numeric"
       data-layout="tenkey">
```

| HTML属性 | JavaScript | 上の例の値 |
| --- | --- | --- |
| `type` | `el.type` | `"text"` |
| `inputmode` | `el.inputMode` | `"numeric"` |
| `data-keyboard-type` | `el.dataset.keyboardType` | `"numeric"` |
| `data-layout` | `el.dataset.layout` | `"tenkey"` |

`dataset` では `data-` を除き、ハイフン直後のASCII小文字を大文字に変換してハイフンを取り除く。たとえば `data-max-length` は `dataset.maxLength`。すべてのハイフンが無条件に消えるわけではない。

属性値は文字列として取得される。`data-enabled="false"` も文字列 `"false"` なので、真偽値として使うなら明示的に比較・変換する。[MDN：dataset](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/dataset)

## 3. C++：QVariantMapから構造体へ変換

```cpp
#include <QDebug>
#include <QString>
#include <QVariant>
#include <QVariantMap>
#include <QWebEnginePage>

struct InputElementInfo
{
    bool editable = false;
    QString type;
    QString inputMode;
    QString keyboardType;
    QString layout;
};

InputElementInfo toInputElementInfo(const QVariantMap &map)
{
    InputElementInfo info;
    info.editable     = map.value(QStringLiteral("editable")).toBool();
    info.type         = map.value(QStringLiteral("type")).toString();
    info.inputMode    = map.value(QStringLiteral("inputMode")).toString();
    info.keyboardType = map.value(QStringLiteral("keyboardType")).toString();
    info.layout       = map.value(QStringLiteral("layout")).toString();
    return info;
}

// jsCodeには「1」のJavaScript全体を設定する。
void queryInputElement(QWebEnginePage *page, const QString &jsCode)
{
    if (!page) return;

    page->runJavaScript(jsCode, [](const QVariant &result) {
        if (!result.isValid() || !result.canConvert<QVariantMap>()) {
            qWarning() << "Failed to obtain input element information";
            return;
        }

        const InputElementInfo info = toInputElementInfo(result.toMap());
        qDebug() << info.editable << info.type << info.inputMode
                 << info.keyboardType << info.layout;
        // ここでinfoを使って仮想キーボードの表示・配列を決める。
    });
}
```

処理の流れは `JavaScript Object → QVariant → QVariantMap → InputElementInfo`。`JSON.stringify()` は不要で、使うと返却値が文字列になる。欠けたキーは、この変換例では `false` または空文字に落ち着く。厳密な入力検証が必要ならキーの存在と型も検査する。

`runJavaScript()` は非同期で、結果はコールバック内で使う。返却対象は通常のデータにし、DOM要素・関数・Promise自体は返さない。ページ破棄時にも無効な値でコールバックが呼ばれ得るため、そこでページやビューを操作しない。この例は `this` をキャプチャしない。[Qt：runJavaScript](https://doc.qt.io/qt-6.5/qwebenginepage.html#runJavaScript)

## 4. フォーカス検知での注意点

- この処理は実行時点の情報を一度取得する。フォーカス変更の自動通知が必要なら、別途 `focusin` などを監視し、QWebChannelなどでC++へ通知する。
- 入力欄が選択されていないと `activeElement` が `body` などになることもある。必ず `editable` で判断する。
- iframe内やShadow DOM内部の入力欄は、このコードだけではたどらない。必要なら対象のフレーム／Shadow Rootで追加取得する。

参考：[MDN：activeElement](https://developer.mozilla.org/en-US/docs/Web/API/Document/activeElement)、[MDN：isContentEditable](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/isContentEditable)
