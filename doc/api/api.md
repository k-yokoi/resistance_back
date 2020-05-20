

# はじめに

本プロジェクトは基本 Websocket の通信であり Path・HTTP メソッド定義が存在せず Swagger ではわかりにくいと思い普通に Markdown で書いた  
API Gateway + Lambda における Websocket の仕組みがわからない場合には以下のドキュメントが有益  
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
並びに、ゲームのステートと API の関係について述べる  


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
  "state": "<次のゲームのステート>"
}
```

ステータスコードはなんとなく HTTP ステータスコードに従う  
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
<td class="org-left">assigning-plots</td>
<td class="org-left">陰謀カードの割り当てを行う</td>
</tr>


<tr>
<td class="org-right">6</td>
<td class="org-left">wait-immediate-effect</td>
<td class="org-left">即時効果(「情報開示」、「責任者」、「立ち聞きされた会話」) の決定を待つ</td>
</tr>


<tr>
<td class="org-right">7</td>
<td class="org-left">assigning-mission</td>
<td class="org-left">リーダーのミッション参加者を決定</td>
</tr>


<tr>
<td class="org-right">8</td>
<td class="org-left">wait-prevoting</td>
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
<td class="org-left">mission-completed</td>
<td class="org-left">ミッションの完了、結果の閲覧待ち</td>
</tr>


<tr>
<td class="org-right">15</td>
<td class="org-left">wait-strong-leader</td>
<td class="org-left">「強力なリーダー」の使用有無を待つ</td>
</tr>
</tbody>
</table>


## ステート図

各ステートでどのような API が実行されるとどのステートに進むのかという話が定義されている  

![API-State](https://github.com/k-yokoi/resistance_back/blob/yoshiki-dev/doc/api/api-state.jpeg)


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

部屋番号を指定し、部屋への参加  


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
  "state": "ready"
}
```


# ゲーム操作系


## assign-role

リーダーが役職を決定の開始を命令する  


### Request

```json
{
  "action": "assign-role",
  "data": {}

}
```


### Response

レスポンスは全員にくる  
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
	],
	"firstLeader": "<最初のリーダーのユーザー ID>"
  },
  "state": "checking-role"
}
```


## start

役職確認後 GM によってゲームの開始を宣言  


### Request

```json
{
  "action": "start",
  "data": {}
  }
}
```


### Response

全員に陰謀カードのリストが配られる  

```json
{
  "status": "200",
  "data": {
	"plots": [ "<陰謀カードの Id>" ]
  },
  "state": "checking-plot"
}
```


## checked-plots

リーダーが実行する  
次のステートをもらうだけ  


### Request

```json
{
  "action": "checked-plots",
  "data": {}
}
```


### Response

「信用の確立」がある場合には、 `wait-trust` に遷移、なければ assign-cards に遷移  

```json
{
  "status": "200",
  "data": {},
  "state": "wait-trust || assigning-plots"
}
```


## execute-tust

「信用の確立」を実行  


### Request

役割カードを開示するユーザー ID を指定  

```json
{
  "action": "execute-trust",
  "data": {
	"trust": "<開示先のユーザーの ID>"
  }
}
```


### Response

全員に以下のレスポンスが届く  
request で指定されたユーザー ID に対しては、 `resistance` or `spy` が見える  
それ以外のユーザーには、 `unknown` が届く  

```json
{
  "status": "200",
  "data": {
	"trust": {
	  "userId": "<リーダーのユーザー ID>",
	  "role": "resistance || spy || unknown"
	}
  },
  "state": "assigning-plots"
}
```


## assign-plots

リーダーが陰謀カードの割り当てを命令する  


### Request

```json
{
  "action": "assign-plots",
  "data": {
	"assign": [
	  {
		"plotId": "<陰謀カードの ID>",
		"userId": "<ユーザーの ID>"
	  },
	]
  }
}
```


### Response

全員に以下のレスポンスが届く  
即時に使用するカードがなければ `assign-mission` へ遷移する  

```json
{
  "status": "200",
  "data": {
	"assigned": [
	  {
		"plotId": "<陰謀カードの ID>",
		"userId": "<ユーザーの ID>"
	  },
	]
  }
  "state": "wait-immediate-effect || assigning-mission"
}
```


## execute-disclosure

「情報開示」の施行  


### Request

```json
{
  "action": "execute-disclosure",
  "data": {
	"disclosure": "<開示先のユーザーの ID>"
  }
}
```


### Response

全員に以下のレスポンスが届く  
request で指定されたユーザー ID に対しては、 `resistance` or `spy` が見える  
それ以外のユーザーには、 `unknown` が届く  

```json
{
  "status": "200",
  "data": {
	"trust": {
	  "userId": "<役割が開示されたユーザーの ID>",
	  "role": "resistance || spy || unknown"
	}
  },
  "state": "wat-immediate-effect || assigning-mission"
}
```


## execute-responsible

「責任者」の施行  


### Request

```json
{
  "action": "execute-responsible",
  "data": {
	"responsible": {
	  "userId": "<引き受けり元のユーザー ID>",
	  "plotId": "<引き取った陰謀カードの ID>"
	}
  }
}
```


### Response

```json
{
  "status": "200",
  "data": {
	"resposible": {
	  "sourceInfo": {
		"userId": "<引き受けり元のユーザー ID>",
		"plotId": "<引き取った陰謀カードの ID>"
	  },
	  "destinationUserId": "<引き受けり先のユーザー ID>"
	}
  },
  "state": "wait-immediate-effect || assigning-mission"
}
```


## execute-eavesdrop

「立ち聞きされた会話」の施行  


### Request

左右が定義されて、指定可能なユーザーかどうかの validation をサーバーサイドで行う  

```json
{
  "action": "execute-eavesdrop",
  "data": {
	"disclosure": "<役割を見たいのユーザーの ID>"
  }
}
```


### Response

全員に以下のレスポンスが届く  
request したユーザーには、 `resistance` or `spy` が見える  
それ以外のユーザーには、 `unknown` が届く  

```json
{
  "status": "200",
  "data": {
	"trust": {
	  "userId": "<役割が開示されたユーザーの ID>",
	  "role": "resistance || spy || unknown"
	}
  },
  "state": "wait-immediate-effect || assigning-mission"
}
```


## assign-mission

リーダーがミッションの参加者を決定  


### Request

```json
{
  "action": "assign-mission",
  "data": {
	"assigned": [ "<ミッションに参加するユーザー ID>" ]
  }
}
```


### Response

```json
{
  "status": "200",
  "data": {
	"assigned": [ "<ミッションに参加するユーザー ID>" ]
  },
  "state": "wait-prevoting || voting"
}
```


## prevote

「総意の形成者」による投票  


### Request

```json
{
  "action": "prevote",
  "data": {
	"vote": "agree || disagree"
  }
}
```


### Response

```json
{
  "status": "200",
  "data": {
	"votes":[
	  {
		"userId": "<投票者のユーザーの ID>",
		"vote": "agree || disagree"
	  },
	]
  },
  "state": "wait-prevoting || voting"
}
```


## vote

一般人による投票  


### Request

```json
{
  "action": "vote",
  "data": {
	"vote": "agree || disagree"
  }
}
```


### Response

```json
{
  "status": "200",
  "data": {
	"votes":[
	  {
		"userId": "<投票者のユーザーの ID>",
		"vote": "agree || disagree"
	  },
	]
  },
  "state": "voting || wait-decline || wait-attention || running-mission || wait-strong-leader || assigning-mission"
}
```


## decide-decline

「不信」使用の意思決定  


### Request

```json
{
  "action": "decide-decline",
  "data": {
	"decline": "true || false"
  }
}
```


### Response

利用する場合としない場合で次に移るステートが違う  

利用する場合には、投票は否決となるので `wait-strong-leader` or `assigning-mission`  

```json
{
  "status": 200,
  "data": {
	"decline": true
  },
  "state": "wait-strong-leader || assigning-mission"
}
```

利用しない場合には、可決なのでミッションが始まる (`wait-attention` or `running-mission`)  

```json
{
  "status": 200,
  "data": {
	"decline": false
  },
  "state": "wait-attention || running-mission"
}
```


## decide-attention

「注目の的」使用の意思決定  


### Request

```json
{
  "action": "decide-decline",
  "data": {
	"attention": "<ユーザー ID> || None"
  }
}
```


### Response

```json
{
  "status": 200,
  "data": {
	"attention": "<ユーザー ID> || None"
  },
  "state": "running-mission"
}
```


## deliver-mission

ミッション参加者によるミッションカード選択  


### Request

```json
{
  "action": "decide-decline",
  "data": {
	"deliver": "true || false"
  }
}
```


### Response

```json
{
  "status": 200,
  "data": {},
  "state": "running-mission || wait-watcher || mission-completed"
}
```


## decide-watcher

「監視者」使用の意思決定  


### Request

```json
{
  "action": "decide-decline",
  "data": {
	"watch": "<ユーザー ID> || None"
  }
}
```


### Response

```json
{
  "status": 200,
  "data": {
	"watch": "<ユーザー ID> || None"
  },
  "state": "mission-completed"
}
```


## mission-result

ミッション結果の計算  


### Request

```json
{
  "action": "mission-result",
  "data": {}
}
```


### Response

次のステートによって 2 種類に別れる  
`wait-strong-leader` の場合  

```json
{
  "status": 200,
  "data": {
	"missionSuccess": "true || false"
  },
  "state": "wait-strong-leader"
}
```

`checking-plot`  

```json
{
  "status": 200,
  "data": {
	"next-leader": "<ユーザー ID>",
	"plots": [ "<陰謀カードの Id>" ],
	"missionSuccess": "true || false"
  },
  "state": "checking-plot"
}
```

ここ少し気持ち悪いからステート増やしてもいいかも？  


## decide-strong-leader

「強力なリーダー」使用の意思決定  


### Request

```json
{
  "action": "decide-strong-leader",
  "data": {
	"strong-leader": "true || false"
  }
}
```


### Response

```json
{
  "status": 200,
  "data": {
	"next-leader": "<ユーザー ID>",
	"plots": [ "<陰謀カードの Id>" ]
  },
  "state": "checking-plot"
}
```

