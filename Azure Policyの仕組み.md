# ポリシーの仕組みについて

## 1. 目的とゴール
ポリシーの仕組みを理解し、ポリシーの読み取りや、カスタムポリシー作成に活かすことができる

## 2. 概要
AzureポリシーはAzureでの<span style="color: red;">**リソース**</span>作成・更新内容をポリシールールと照合し、許可や拒否などを行う仕組み

> **※重要**  
例えば、「ストレージアカウント」と、ストレージアカウントで設定する「ライフサイクル管理ルール」はAzure上では<span style="color: red;">**別リソース**</span>として扱われるため注意

<span style="color: green;">※ストレージアカウント系が私のトレンドであるため、例がだいたいストレージアカウント関係となっております。ご了承ください。</span>

## 3. ポリシーの評価対象

### 3-1. ポリシーは何を見ているのか
ポリシーは基本的に、現在作成または更新されようとしている<span style="color: red;">**リソース**</span>を見る

例えば、ストレージアカウントを作成する場合は、以下のリソースをポリシーは見る
```
Microsoft.storage/storateAccounts
```

ライフサイクル管理を保存する場合は、以下の子リソースをポリシーは見る
```
Microsoft.storage/storateAccounts/managementPolicies
```

Azureポータルでは同じストレージアカウント画面から操作するため、同じリソースを変更しているように見えるが、
Azure Rexouuce Managerでは別リソースとして扱われる
```
ストレージアカウント
Microsoft.storage/storateAccounts
│
└── ライフサイクル管理
　　type：Microsoft.storage/storateAccounts/managementPolicies
　　name：default
    Properties：※ルール名等が含まれる（Policy.rules）
```

<span style="color: green;">**Tips**</span>   
<span style="color: green;">ライフサイクル管理の名前は 'default' にする必要がある</span>  
<span style="color: green;">↓公式ページ</span>  
https://learn.microsoft.com/ja-jp/rest/api/storagerp/management-policies/create-or-update?view=rest-storagerp-2025-08-01&utm_source=chatgpt.com&tabs=HTTP


### 3-2. Type
ポリシーの条件「type」で評価対象のリソースを絞っている

つまり、  
ポリシーで<span style="color: purple;">**ライフサイクル管理**</span> の最低日数を定めたい場合  
１）ポリシーの「type」＝「Microsoft.storage/storateAccounts」の場合  
→ポリシーが適用されない  

２）ポリシーの「type」＝「Microsoft.storage/storateAccounts/<span style="color: purple;">**managementPolicies**</span>」の場合  
→ポリシーが適用される  



Value
**field**がリソースの項目を直接見るのに対し、  
**value**は関数を使って値を計算する

例えば、
```
{
    "value":"[resourceGroup().name]
    "like":"rg-*-storage"
}
```
↑<span style="color: red;">**評価対象リソース**</span> が所属するリソースグループ名を取得し、上記の条件と一致するか判定

今回の<span style="color: red;">**評価対象リソース**</span>がライフサイクル管理の場合  
"[resourceGroup().name]の結果は  
以下のリソースIDのリソースグループ名が返ってくる 
●リソースID  
```
/subscription/XXX(サブスクリプション名)
/resourceGroup/XXX(リソースグループ名)
/providers/Microsoft.storage
/storageAccounts/xxx(ストレージアカウント名)
/managementPolicies/default(ライフサイクル管理名は常にdefault)
```
