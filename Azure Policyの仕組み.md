# ポリシーの仕組みと「field」/「value」について

## 1. 目的とゴール
ポリシーの仕組みを理解し、ポリシーの読み取りや、カスタムポリシー作成に活かすことができる  
ポリシー規則の「field」と「value」の違いを理解することができる

## 2. 仕組みの概要
AzureポリシーはAzureでの<span style="color: red;">**リソース**</span>作成・更新内容をポリシールールと照合し、許可や拒否などを行う仕組み

> **※重要**  
例えば、「ストレージアカウント」と、ストレージアカウントで設定する「ライフサイクル管理ルール」はAzure上では<span style="color: red;">**別リソース**</span>として扱われるため注意

<span style="color: green;">※ストレージアカウント系が私のトレンドであるため、例がだいたいストレージアカウント関係となっております。ご了承ください。</span>

## 3. ポリシーは何を見ているのか（ポリシーの評価対象）

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
Azure Resouce Managerでは別リソースとして扱われる
```
ストレージアカウント
Type:Microsoft.storage/storateAccounts
name:Storagename（自由に決めることができる）
│
└── ライフサイクル管理
　　type：Microsoft.storage/storateAccounts/managementPolicies
　　name：default（すでに決められている・変更不可）
    Properties：※ルール名等が含まれる（Policy.rules）
    │
    └── ルール
        name:Rulename（自由に決めることができる）
    　　type:Lifecycle
```

### Tips   
<span style="color: green;">ライフサイクル管理の名前は 'default' にする必要がある</span>  
<span style="color: green;">↓公式ページ</span>  
https://learn.microsoft.com/ja-jp/rest/api/storagerp/management-policies/create-or-update?view=rest-storagerp-2025-08-01&utm_source=chatgpt.com&tabs=HTTP

## 4. field
field役割は、Azureリソースが持っているプロパティを直接指定して、その値をチェックすること

参照ページ：https://learn.microsoft.com/ja-jp/azure/governance/policy/concepts/definition-structure-policy-rule#field-expressions

## 4-1. type
ポリシーの条件「type」で**評価対象のリソース**を絞っている

つまり、例えば  
ポリシーで<span style="color: purple;">**ライフサイクル管理**</span> の最低日数を定めたい場合  
１）ポリシーの「type」＝「Microsoft.storage/storateAccounts」の場合  
→ポリシーが適用されない  

２）ポリシーの「type」＝「Microsoft.storage/storateAccounts/<span style="color: purple;">**managementPolicies**</span>」の場合  
→ポリシーが適用される  

### ポイント
Azure Policyは、リソースの作成・更新時に、その操作の対象となる**リソース**を評価する。  

今回のように、**ライフサイクル管理**の設定内容（ルールや日数など）を評価したい場合は、**ライフサイクル管理**自体の新規作成・更新をきっかけに評価する必要がある。  

そのため、**type** は**ライフサイクル管理**のリソース型を指定する必要がある。  

一方、**type** を**ストレージアカウント**に指定しても、今回変更しているのは**ライフサイクル管理**の設定であり、<ins>**ストレージアカウント**自体の情報は変更されていない。<ins>  
そのため、**ストレージアカウント**を評価対象にしても、変更された**ライフサイクル管理**の設定内容を適切に評価できない。  

つまり、<ins>評価したい設定が属しているリソースの種類を type に指定することが重要。<ins>  


## 4-2. id
**id**はリソース」ID。  
**id** はリソースの「一意の識別パス（住所）」を指し、field: **type** はリソースの「種類（カテゴリ）」を指します。


| field | 値の例 |
| --- | --- |
| id | /subscriptions/xxxx/resourceGroups/rg1/providers/Microsoft.Compute/virtualMachines/vm01 |
| type | Microsoft.Compute/virtualMachines |


## 4-3. name/fullname
Azure Policyにおける **name** と **fullName** の違いは、リソースの名前だけを取得するか、親リソースを含めた完全な名前を取得するか という違い。
| Field	| 意味 | 取得する値
| --- | --- | --- |
| name | リソース自身の名前 | default
| fullName | 親リソースを含めた完全な名前 | storageaccount01/default


例えば、対象リソース(Typeで指定しているリソース)がライフサイクル管理の場合、以下のようになる

```
name＝default
fullName＝Storagename(ストレージアカウント名)/default
```
※fullname は親子関係を含めた名前であり、リソースIDはさらに上位の階層まで含めた識別情報である。  
　そのためfullNameはストレージアカウント名/defalutという形になる  
参照ページ：https://learn.microsoft.com/ja-jp/azure/governance/policy/concepts/definition-structure-policy-rule#fields



## 5. value
valueの役割は指定した値や関数の計算結果を取得して、条件と比較すること  
fieldと比較すると、  
**field**がリソースの項目を直接見るのに対し、  
**value**は関数を使って値を計算する

## 5-1. [first(split(field('fullName'),'/'))]


ライフサイクル管理のfullNameを
```
split(field('fullName'),'/'
```
すると以下のようになる
```
[
    "Storagename(ストレージアカウント名)"
    "default"
]
```

つまり↓下記は「fullName」の先頭部分、つまりストレージアカウント名だけを取り出す  
＝ポリシーでライフサイクル管理に紐づくストレージアカウント名に条件を付けることができる 

```
[first(split(field('fullName'),'/'))]
```
 
※ 例：
```
{
    "value":"[first(split(field('fullName'),'/'))]"
    "like":"st-*-hitsuji"
}
```
first関数参照ページ：  
https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/template-functions-array#first



## 5-2. [resourceGroup().name]

例えば、
```
{
    "value":"[resourceGroup().name]"
    "like":"rg-*-arupaka"
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

resourceGroup関数参照ページ：  
https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/template-functions-scope#resourcegroup


関数一覧ページ：  
https://learn.microsoft.com/ja-jp/azure/azure-resource-manager/templates/template-functions
