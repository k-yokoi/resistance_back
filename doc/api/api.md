

# はじめに

Swagger にしようと思ったが、本プロジェクトは基本 Websocket の通信であり Path・HTTP メソッド定義が存在せず Swagger ではわかりにくいと思い普通に Markdown にする  
そもそも、どんな API Gateway + Lambda における Websocket における仕組みがわからない場合には以下のドキュメントが有益  
< https://dev.classmethod.jp/articles/api-gateway-websocket-serverless/ >  

HTTP では、ユーザーの情報が欲しければ < http://endpoint.com/api/get-user?id=yoshiki > のような形でリクエストを行うが、  
API Gateway + Lambda では、Websocket では確立したセッションに対して以下のようにリクエストを行う  

```json
{
  "action": "get-user",
  "data": {
	"query": "yoshiki"
  }
}
```

`action` に指定した値によってバックエンドで行われる処理が定義される  
`data` の中に、 `action` で指定した処理で必要なデータが保存されている  

そのため、このドキュメントではどういう `action` があるのか、その `action` に対してどういうパラメータが必要なのかを定義する  


# ゲーム管理系


## create

部屋作成のためのアクション  


### Request

```json
{
  "action": "create"
}
```


### Response

```json
{
  "status": "200",
  "data": {
	"roomId": "<部屋番号>",
	"userId": "<部屋主の User ID>"
  }
}
```


## join


### Request

```json
{
  "action": "join",
  "data": {
	"query": "<number room>"
  }
}
```


### Response

部屋全員に以下のレスポンスが返却される  

```json
{
  "status": "200",
  "data": {
	"userId": "<join した参加者の User ID>"
  }
}
```


## get-user


# ゲーム操作系


## start


### Request

```json
{
  "action": "start"
  "data": {
	"roomId": "<部屋番号>"
  }
}
```


### Response

全員にレスポンスがくる  
レジスタンス側には `role` が `unknown` のレスポンスがきます  
スパイ側は全員の `role` が記載されたレスポンスがきます  

```json
{
  "status": "200",
  "data": {
	"roles": [
	  {
		"userId": "<ユーザーの ID>",
		"userName": "<ユーザーの名前>",
		"role": "resistance || spy || unknown"
	  },
	  ...
	]
  }
}
```

**\***  

