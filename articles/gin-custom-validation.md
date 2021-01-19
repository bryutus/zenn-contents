---
title: "Ginフレームワークでのカスタムバリデーション"
emoji: "🍹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go, gin]
published: true
---

Ginではデフォルトのバリデーターとして [go-playground/validator]( https://github.com/go-playground/validator ) が使用されています。
国コード等の様々なルールが存在しているのですが、言語コードは用意されていなかったため、カスタムバリデーションを作成しました。

## バージョン

- **go**: 1.14.3
- **gin-gonic/gin**: v1.6.3
- **go-playground/validator**: v10.2.0

## Ginでのバリデーション

構造体のタグに指定されたルールに基づいて個々のフィールドの値のバリデーションを行います。
以下の例では `language_code` というクエリパラメータに対して「必須項目、2文字」というルールを設定しています。

```go:main.go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
)

type PartyName struct {
    LanguageCode string `form:"language_code" binding:"required,len=2"`
}

func main() {
    r := gin.Default()

    r.GET("/party_name", getPartyName)
    r.Run()
}

func getPartyName(c *gin.Context) {
    var p PartyName
    if err := c.ShouldBind(&p); err == nil {
        c.JSON(http.StatusOK, gin.H{"message": "PartyName language code is valid!"})
    } else {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    }
}
```

## カスタムバリデーションを追加

言語コード（ISO 639-1 Alpha2）のバリデーションを追加します。
既に国コードのバリデーションは存在していましたので、それを参考にルールを作成しました。
マップの中に対象の言語コードが存在しているか否かを判定しているだけです。

```go
var iso639Alpha2 = map[string]bool{
    // 言語コードは一部抜粋
    "io": true, "is": true, "it": true, "iu": true, "ja": true,
    "xh": true, "yi": true, "yo": true, "za": true, "zh": true,
    "zu": true,
}

var isIso639Alpha2 validator.Func = func(fl validator.FieldLevel) bool {
    val := fl.Field().String()
    return iso639Alpha2[val]
}
```

作成した `isIso639Alpha2` メソッドを、新たなルールとしてバリデーターに登録します。
`PartyName` という構造体のタグに新しく登録したルールを指定することで、 `language_code` というクエリパラメータに対して登録したルールを適用することができます。

```go
type PartyName struct {
    LanguageCode string `form:"language_code" binding:"required,iso639_1_alpha2"`
}

func main() {
    r := gin.Default()

    if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
        v.RegisterValidation("iso639_1_alpha2", isIso639Alpha2)
    }

    r.GET("/party_name", getPartyName)
    r.Run()
}
```

全体のコードはこのようになります。
[公式にサンプル]( https://github.com/gin-gonic/examples/blob/master/custom-validation/server.go )がありましたので、そちらを参考に実装しました。

```go:main.go
package main

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/gin-gonic/gin/binding"
    "github.com/go-playground/validator/v10"
)

type PartyName struct {
    LanguageCode string `form:"language_code" binding:"required,iso639_1_alpha2"`
}

var iso639Alpha2 = map[string]bool{
    // 言語コードは一部抜粋
    "io": true, "is": true, "it": true, "iu": true, "ja": true,
    "xh": true, "yi": true, "yo": true, "za": true, "zh": true,
    "zu": true,
}

var isIso639Alpha2 validator.Func = func(fl validator.FieldLevel) bool {
    val := fl.Field().String()
    return iso639Alpha2[val]
}

func main() {
    r := gin.Default()

    if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
        v.RegisterValidation("iso639_1_alpha2", isIso639Alpha2)
    }

    r.GET("/party_name", getPartyName)
    r.Run()
}

func getPartyName(c *gin.Context) {
    var p PartyName
    if err := c.ShouldBind(&p); err == nil {
        c.JSON(http.StatusOK, gin.H{"message": "PartyName language code is valid!"})
    } else {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    }
}
```

## カスタムバリデーションのテスト

作成したルールのテストはこのように書きました。
`Var` メソッドを使用して単一の変数 `test.in` の値をバリデーターに登録したルールを用いて検証しています。

```go:main_test.go
package main

import (
    "testing"

    "github.com/go-playground/validator/v10"
)

func TestIsIso639Alpha2(t *testing.T) {
    testCases := []struct {
        in   string
        want bool
    }{
        {"is", true},
        {"isl", false},
        {"qq", false},
    }

    validate := validator.New()
    validate.RegisterValidation("iso639_1_alpha2", isIso639Alpha2)

    for i, test := range testCases {
        errs := validate.Var(test.in, "iso639_1_alpha2")
        if test.want {
            if errs != nil {
                t.Fatalf("Index: %d iso639_1_alpha2 failed Error: %s", i, errs)
            }
        } else {
            if errs == nil {
                t.Fatalf("Index: %d iso639_1_alpha2 failed Error: %s", i, errs)
            }
        }
    }
}
```

---

色々なユースケースのサンプルをまとめた[Ginのリポジトリ]( https://github.com/gin-gonic/examples )がありましたので、カスタムバリデーション以外でも実装する機会に参考にしてみると良いかもしれません。
