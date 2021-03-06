# Davai [![GoDoc](https://godoc.org/github.com/teacat/davai?status.svg)](https://godoc.org/github.com/teacat/davai) [![Coverage Status](https://coveralls.io/repos/github/teacat/davai/badge.svg?branch=master)](https://coveralls.io/github/teacat/davai?branch=master) [![Build Status](https://travis-ci.org/teacat/davai.svg?branch=master)](https://travis-ci.org/teacat/davai) [![Go Report Card](https://goreportcard.com/badge/github.com/teacat/davai)](https://goreportcard.com/report/github.com/teacat/davai)

基於 `net/http` 的 Golang 基本 HTTP 路由，這個套件試圖提供最核心且具動態路由功能的路由器。

# 這是什麼？

Davai（давай）是一個十分快速的 HTTP 路由器，這能夠讓你有效地作為其它套件的基礎核心。

-   支援中介軟體（Middleware）。
-   極具動態的路由擷取、前後輟功能，亦透過正規表達式快取提升效能。
-   相容 `net/http` 的原生用法而無需重新學習。
-   反向路由能從變數產生網址。
-   路由群組以避免繁雜的重複手續。
-   能夠自行更改路由優先度。
-   支援靜態檔案服務輸出。

# 為什麼？

多數的 Golang 路由器以 `:name` 作為路徑擷取的用法，但這種用法導致你不能夠擁有固定前後輟（例如：`user-:id.json`），而 Davai 解決了這個問題，且讓路由擷取變得更加多元化、也更適合 RESTful API 設計。

在不少路由器比對路徑時，都會從上搜尋到下（即為：從多到少）。就算是個最基本的 `/` 路徑都必須先從最長且可能帶有正規表達式的動態路由開始探索，這浪費了不少的時間，而 Davai 將動態和靜態路由區分進而減少效能損耗。

額外一點在於 Davai 能夠快取部分網址來避免重複的正規表達式驗證、且相容原生的 `net/http` 函式讓使用設計更加地通用。

# 效能比較

這裡有份簡略化的[效能測試報表](https://github.com/teacat/davai-benchmark)。目前仍會持續優化並且增加快取以避免重複分析先前路由而費時。

```
測試規格：
1.7 GHz Intel Core i7 (4650U)
8 GB 1600 MHz DDR3

雙重擷取群組選擇： Gramework > HTTPRouter > Davai > HTTPTreeMux > Gin > Pat > Mux > Echo > Beego
21583.24 (reqs/s) | davai, /user/yamiodymel/admin
22005.52 (reqs/s) | gramework, /user/yamiodymel/admin
21632.75 (reqs/s) | httprouter, /user/yamiodymel/admin
18313.46 (reqs/s) | martini, /user/yamiodymel/admin
19869.14 (reqs/s) | pat, /user/yamiodymel/admin
21218.12 (reqs/s) | gin, /user/yamiodymel/admin
19548.15 (reqs/s) | mux, /user/yamiodymel/admin
21354.90 (reqs/s) | httptreemux, /user/yamiodymel/admin
21005.71 (reqs/s) | echo, /user/yamiodymel/admin
17983.01 (reqs/s) | beego, /user/yamiodymel/admin
```

![](./assets/screenshot.png)

# 索引

* [安裝方式](#安裝方式)
* [使用方式](#使用方式)
    * [變數路由](#變數路由)
    * [選擇性路由](#選擇性路由)
	* [前後輟路由](#前後輟路由)
	* [任意路由](#任意路由)
    * [正規表達式路由](#正規表達式路由)
        * [自訂規則](#自訂規則)
	* [路由優先度](#路由優先度)
    * [路由群組](#路由群組)
    * [反向與命名路由](#反向與命名路由)
    * [中介軟體](#中介軟體)
		* [進階建構體](#進階建構體)
		* [群組區域](#群組區域)
		* [路由器區域](#路由器區域)
    * [提供靜態資源與目錄](#提供靜態資源與目錄)
		* [單個檔案](#單個檔案)
		* [允許目錄索引](#允許目錄索引)
    * [無路由](#無路由)
	* [良好結束](#良好結束)
* [如何運作的？](#如何運作的)


# 安裝方式

打開終端機並且透過 `go get` 安裝此套件即可。

```bash
$ go get github.com/teacat/davai
```

# 使用方式

透過 `davai.New` 建立一個新的路由器，並且以 `Get`、`Post`⋯ 等方法來建立基於不同路由的處理函式，接著將路由器傳入 `http.Handle` 來開始監聽服務。

```go
package main

import (
	"net/http"

	"github.com/teacat/davai"
)

func main() {
	d := davai.New()
	d.Get("/", func(w http.ResponseWriter, r *http.Request) {
		// ...
	})
	d.Get("/posts", func(w http.ResponseWriter, r *http.Request) {
		// ...
	})
	d.Post("/album", func(w http.ResponseWriter, r *http.Request) {
		// ...
	})
	d.Run()
}
```

## 變數路由

透過 `{}`（花括號）符號可以擷取路由中指定片段的內容並作為指定變數在路由器中使用。

```
路由：/user/{name}

/user/admin                ○
/user/admin/profile        ✕
/user/                     ✕
```

在路由中以 `davai.Vars` 並傳入 `*http.Request` 來取得在路由中所擷取的變數。

```go
func main() {
	d := davai.New()
	d.Get("/post/{title}", func(w http.ResponseWriter, r *http.Request) {
		// 透過 `davai.Vars` 並傳入 HTTP 請求的建構體就能夠取得已擷取的變數。
		vars := davai.Vars(r)
		// 存取 `vars` 來取得網址中的變數。
		// 如果該變數是選擇性的，在沒有該變數的情況下會是一個空字串值。
		fmt.Println(vars["title"])
	})
	// ...
	d.Run()
}
```

## 選擇性路由

如果擷取的變數並不一定是必要的，那麼可以在變數名稱後加上 `?` 來作為「選擇性變數」。

```
路由：/user/{name?}

/user/                     ○
/user/admin                ○
/user/admin/profile        ✕
```

## 前後輟路由

擷取路由的前、後可以參雜靜態文字，這讓你很好設計一個基於 `.json` 副檔名的 RESTful API 系統。

```
路由：/api/resource-{id}.json

/api/resource-123.json     ○
/api/resource-.json        ✕
/api/                      ✕
```

## 任意路由

透過 `*` 規則可以讓正規表達式符合任何型態的路徑。當這個規則被擺放在路由的最後面時即會成為「任意路由」，在這種情況下任何路徑都會符合。

```
路由：/src/{*:filename}

/src/                      ○
/src/example.png           ○
/src/subdir/example.png    ○
/api/                      ✕
```

## 正規表達式路由

透過正規表達式路由可以更精準地表明路由應該要符合哪種格式，Davai 預設有數種正規表達式規則：`i`（數字）、`s`（數字與英文字母）。

在變數路由名稱的前面加上 `:` 來表明欲使用的正規表達式規則，其格式為 `{規則:變數名稱}`。用上正規表達式後亦能在變數名稱後加上 `?`（問號）來作為選擇性路由。

```
路由：/user/{i:id}

/user/1234                 ○
/user/                     ✕
/user/profile              ✕
/user/1234/profile         ✕
```

### 自訂規則

如果 Davai 預設的正規表達式規則不合乎你的需求，可以考慮透過 `Rule` 來追加新的正規表達式規則。

```go
func main() {
	d := davai.New()
	// 透過 `Rule` 可以追加新的正規表達式規則。
	d.Rule("r", "[0-9a-z]++")
	// 接著就能夠直接在路由中使用。
	d.Get("/post/{r:name}", PostHandler)
	// ...
	d.Run()
}
```

## 路由優先度

如果有些路由希望能夠優先執行，那就可以透過 `AddPriority` 來將其提昇優先度。優先度的運作規則請參閱「[如何運作的？](#如何運作的)」章節。

```go
func main() {
	d := davai.New()
	// 這原本會被另一個路由優先取代，但透過提昇優先度之後就能夠先執行這個路由。
	d.Get("/{language}", myHandler).AddPriority(100)
	// 由於優先度被變更了，現在這個路由會在上面之後才執行。
	d.Get("/{language}/{*:path}", myAnotherHandler)
	d.Run()
}
```

## 路由群組

如果有些路由的前輟、中介軟體是一樣的話那麼就可以建立一個路由群組來省去重複的手續。

```go
func main() {
	d := davai.New()
	// 透過 `Group` 可以替路由建立一個通用的前輟。
	// 路由群組也能夠有多個，這很適合用在設計 API 並區分版本上。
	v1 := d.Group("/v1")
	{
		v1.Post("/", MyHandler)
		v1.Post("/user", UserHandler)
		v1.Post("/logout", LoginHandler)
	}
	// ...
	d.Run()
}
```

## 反向與命名路由

替定義好的路由命名，就能夠在稍後透過此名稱並傳入變數來反向產生該路由。

```go
func main() {
	d := davai.New()
	// 在路由後面透過 `Name` 函式來替定義好的路由命名。
	d.Get("/product/{type}/{id}", ProductHandler).Name("Product")
	// 透過 `Generate` 可以傳入變數並產生已命名的路由。
	path := d.Generate("Product", map[string]string{}{
		"type":  "large",
		"id":    "152",
	})
	// 結果：/product/large/152
	fmt.Println(path)
}
```

## 中介軟體

中介軟體也稱作中介層，這能夠在單個路由中執行多個處理函式並串在一起。

```go
func MyMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// ...
		// 呼叫 `net.ServeHTTP` 來呼叫下一個中介軟體或者是處理函式。
		// 如果不這麼做的話則會中斷繼續。
		next.ServeHTTP(w, r)
	})
}

func main() {
	d := davai.New()
	// 將 `MyMiddleware` 中介軟體安插於路由中。
	d.Get("/post", MyMiddleware, UserHandler)
	d.Get("/album", MyMiddleware, AlbumHandler)
	// ...
	d.Run()
}
```

### 進階建構體

某些情況下中介軟體的功能可能會非常繁雜，而這個時候單函式設計的中介軟體很難派上用場。此時可以考慮以建構體的方式實作 Davai 中的 `middleware` 介面。

```go
// MyMiddleware 是個會實作 Davai 中介軟體介面的自訂進階中介軟體。
type MyMiddleware struct {
	// ...
}

// Middleware 是實作 Davai 中介軟體介面必須的函式，在這裡會接收下一個中介軟體、處理函式來讓整個函式鏈繼續下去。
func (m *MyMiddleware) Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		next.ServeHTTP(w, r)
	})
}

// MyFunction 是自訂的函式，隨便你想怎麼使用都行。
func (m *MyMiddleware) MyFunction() {
	// ...
}
```

### 群組區域

如果一個中介軟體會用於很多個路由，那麼可以考慮替他們建立一個路由群組並將中介軟體安插於該群組中。

```go
func main() {
	d := davai.New()
	// 中介軟體可以安插於路由群組，這樣就不需要替每個路由安插一個中介軟體。
	v1 := d.Group("/v1", MyMiddleware)
	{
		v1.Post("/user", UserHandler)
		v1.Post("/login", LoginHandler)
	}
	// 事後也可以透過 `Use` 來替群組追加另一個中介軟體。
	v1.Use(AnotherMiddleware)
	// ...
	d.Run()
}
```

### 路由器區域

透過路由的 `Use` 可以安裝全域中介軟體，這會在所有路由中執行。

```go
func main() {
	d := davai.New()
	// 將多個中介軟體作為全域用途並在所有路由中執行。
	d.Use(MyMiddleware, AnotherMiddlewate)
	// ...
	d.Run()
}
```

## 提供靜態資源與目錄

Davai 已經幫助你完善了基本的檔案提供函式，這會呼叫內建的 `http.FileServer` 來處理檔案服務。

透過 `ServeFiles` 來提供靜態目錄，這會友好地幫助你透過 `301` 重新導向那些忘記加後斜線的網址，還能在必要時刻回傳 `403` 錯誤來隱藏目錄下的檔案。

```go
func main() {
	d := davai.New()
	d.ServeFiles("/", "resources")
	// ...
	d.Run()
}
```

### 單個檔案

透過 `ServeFile` 可以提供單個靜態檔案。

```go
func main() {
	d := davai.New()
	d.ServeFile("/file.txt", "resources/file.txt")
	// ...
	d.Run()
}
```

### 允許目錄索引

預設的情況下為了安全起見，Davai 會禁止伺服器暴露目錄下的所有檔案。如果你不希望這麼做的話，在透過 `ServeFiles` 建立路由時，更改其 `DirectoryListing` 設置為 `true` 即可。

```go
func main() {
	d := davai.New()
	// 允許 `/` 目錄暴露所有底下的檔案。
	d.ServeFiles("/", "resources").DirectoryListing = true
	// ...
	d.Run()
}
```

## 無路由

以 `NoRoute` 來傳入一個處理函式，這個函式會在客戶端呼叫了不存在的路由時所執行。

```go
func main() {
	d := davai.New()
	d.NoRoute(NoRouteHandler)
	// ...
	d.Run()
}
```

## 良好結束

Davai 支援良好結束（Graceful Shutdown），這能夠讓你不需要中斷程式就能結束並關閉 Davai 路由器的運作。

```go
	d := davai.New()
	// ...
	go func() {
		<-time.After(time.Second * 30)
		d.Shutdown(context.Background())
	}()
	// ...
	d.Run()
```

# 如何運作的？

Davai 有兩個路由樹，分別是「靜態」與「動態」。當定義一個路由的時候，Davai 會事先解析該路由中的所有片段，並且轉換成靜態資料以提升後續的查詢效能。當接收到一個請求的時候，Davai 會優先找尋靜態路由中是否有符合此路徑的路由，若無則會逐一搜尋動態路由並且相互比對。

每個路由樹都有多個路由並且依照優先程度進行排列，簡單地說：「越長的路由就越會先被執行」。優先順序的定義以路由中的每個片段加總為主：

```
優先度    定義
20       根目錄
16       路徑片段
8        靜態路徑
4        擷取群組
2        帶前後輟
1        正規表達式
-1       可選片段
-2       任意片段
```

這有點類似 Unix 的權限（`1`、`2`、`4`）計算方式，也因為越長的路由會有更多分數，所以就會先被執行、比對；而越怠惰的路由分數就會越低。實際的路由優先順序程度如下：

```
優先度    路由
69       /user/{s:name}/profile
48       /user/admin
24       /user
20       /
19       /{*:path}
```

靜態路由的比對十分簡單，就是直接透過 `map` 的鍵值來比對是否有此路由。然而動態路由則不同，動態路由會將網址拆分成多個片段，並且將其從左至右、從長到短逐一比對，並且判斷下一個片段是否為可選片段，若是可選片段且網址已經到底的話就算符合（因為可選）。

然而 Davai 也支援路由片段的前後輟，這個判別方法十分簡單和迅速。定義路由時 Davai 會事先分析路由片段的前後輟為何，並且保存起來供之後比對請求網址。

```
   片段                  片段
+--------+------------+---------+--------+
|        |     前輟    | 擷取群組 |   後輟  |
+--------+============+---------+========+
/  user  /  resource- |   {id}  | .json  /
+--------+============+---------+========+
```

比對時 Davai 會透過 `strings.HasPrefix` 和 `strings.HasSuffix` 函式來確認該路由是否有指定的前後輟，並且將接收到的網址片段透過 `strings.TrimLeft` 與 `strings.TrimRight` 來移除前後輟，最終就能取得路由中的擷取群組內容。
