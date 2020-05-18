

# はじめに

Swagger にしようと思ったが、本プロジェクトは基本 Websocket の通信であり Path・HTTP メソッド定義が存在せず Swagger ではわかりにくいと思い普通に Markdown にする  
そもそも、どんな API Gateway + Lambda における Websocket における仕組みがわからない場合には以下のドキュメントが有益  
< https://dev.classmethod.jp/articles/api-gateway-websocket-serverless/ >  

HTTP では、ユーザーの情報が欲しければ < http://endpoint.com/api/get-user?id=yoshiki > のような形でリクエストを行うが、  
API Gateway + Lambda では、Websocket では確立したセッションに対して、文字列でリクエスト/レスポンスのやりとりを行う  
例えば、 JSON で以下のように返却される  

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


# 前提

リクエスト/レスポンスは以下のテンプレートに従う  


## リクエスト

```json
{
  "action": "<String: 行いたいアクション>",
  "data": "<JSON: アクションに必要なデータ>"
}
```


## レスポンス

```json
{
  "status": "<INT: ステータスコード>",
  "data": "<JSON: アクションの結果>",
  "state": "<現在のゲームのステート>"
}
```

ステータスコードは HTTP ステータスコードに従う  
フロントエンドは `state` の部分を参照してゲームが今どのフェーズにいるのかを確認してほしい  


## ステート

ステートは以下の表にある通りの名前/意味  

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-right" />

<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<thead>
<tr>
<th scope="col" class="org-right">&#xa0;</th>
<th scope="col" class="org-left">State</th>
<th scope="col" class="org-left">Description</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-right">1</td>
<td class="org-left">ready</td>
<td class="org-left">ユーザーへの参加とゲーム確定 (役割配布) 命令を待つ</td>
</tr>


<tr>
<td class="org-right">2</td>
<td class="org-left">checking-role</td>
<td class="org-left">役割配布後にゲーム開始を待つ</td>
</tr>


<tr>
<td class="org-right">3</td>
<td class="org-left">checking-plot</td>
<td class="org-left">リーダーに陰謀カードを配布</td>
</tr>


<tr>
<td class="org-right">4</td>
<td class="org-left">wait-trust</td>
<td class="org-left">「信用の確立」の決定を待つ</td>
</tr>


<tr>
<td class="org-right">5</td>
<td class="org-left">assigning-cards</td>
<td class="org-left">陰謀カードの割り当てを待つ</td>
</tr>


<tr>
<td class="org-right">6</td>
<td class="org-left">immediate-effect</td>
<td class="org-left">即時効果(「情報開示」、「責任者」、「立ち聞きされた会話」) の決定を待つ</td>
</tr>


<tr>
<td class="org-right">7</td>
<td class="org-left">assigning-mission</td>
<td class="org-left">リーダーのミッション参加者を決定</td>
</tr>


<tr>
<td class="org-right">8</td>
<td class="org-left">prevoting</td>
<td class="org-left">「総意の形成者」の投票を待つ</td>
</tr>


<tr>
<td class="org-right">9</td>
<td class="org-left">voting</td>
<td class="org-left">全員の投票を待つ</td>
</tr>


<tr>
<td class="org-right">10</td>
<td class="org-left">wait-decline</td>
<td class="org-left">「不信」の使用有無を待つ</td>
</tr>


<tr>
<td class="org-right">11</td>
<td class="org-left">wait-attention</td>
<td class="org-left">「注目の的」の使用有無を待つ</td>
</tr>


<tr>
<td class="org-right">12</td>
<td class="org-left">running-mission</td>
<td class="org-left">ミッション参加者の決定を待つ</td>
</tr>


<tr>
<td class="org-right">13</td>
<td class="org-left">wait-watcher</td>
<td class="org-left">「監視者」の使用有無を待つ</td>
</tr>


<tr>
<td class="org-right">14</td>
<td class="org-left">wait-strong-leader</td>
<td class="org-left">「強力なリーダー」の使用有無を待つ</td>
</tr>
</tbody>
</table>


# ゲーム管理系


## create

部屋作成のためのアクション  


### Request

```json
{
  "action": "create",
  "data": {
	"userName": "<部屋主のユーザー名>"
  }
}
```


### Response

```json
{
  "status": "200",
  "data": {
	"roomId": "<部屋番号>",
	"userId": "<部屋主の User ID>"
  },
  "state": "ready"
}
```


## join


### Request

```json
{
  "action": "join",
  "data": {
	"query": "<部屋番号>",
	"userName": "<部屋主のユーザー名>"
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
  },
  "state": "state"
}
```


## get-user


# ゲーム操作系


## assign-role


### Request

```json
{
  "action": "assign-role"
  "data": {
	"roomId": "<部屋番号>"
  }
}
```


### Response

全員にレスポンスがくる  
レジスタンス側には `role` が `unknown` のレスポンスが来る  
スパイ側は全員の `role` が記載されたレスポンスが来る  

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
	],
	"firstLeader": "<最初のリーダーのユーザー ID>"
  },
  "state": "checking-role"
}
```


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
	],
	"firstLeader": "<最初のリーダーのユーザー ID>"
  }
  "state": "cards"
}
```


## distribute-cards


# ステート管理

ユーザー側からの明示的なリクエストは発生しないが、全員が同一の画面を表示させるために  

