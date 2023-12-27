# 1つのリソースで複数テナント用にデータソースを分けて構築する方法

## 背景
Azure AI Search リソースは up にしているだけで月5万程度の金額がかかってしまうため、1つのリソースで複数テナント用に提供したい

## 前提
下記手順は、原則 Japan East での構築を想定
**ただし、`gpt-4 (1106-preview)`, `gpt-4 (vision-preview)` は Japan East で使用不可**  
**また Vectorize API も Japan East で使用不可のため、Embedding 含めた Azure OpenAI と Computer Vision は Japan East 以外を指定する必要がある**  
※ 2023-12-21 現在  
[Azure OpenAI Service モデル#GPT-4 および GPT-4 Turbo プレビュー モデルの可用性](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/concepts/models#gpt-4-and-gpt-4-turbo-preview-model-availability)  
[マルチモーダル埋め込みを使用して画像を取得する (バージョン 4.0 プレビュー)#前提条件](https://learn.microsoft.com/ja-jp/azure/ai-services/computer-vision/how-to/image-retrieval#prerequisites)

データソースが論理的に分かれているため、検索の過程においてはあるテナントが違うテナントのリソースを検索してしまうことはない  
**ただし、データソースとして提示される URL は public のため、URL さえ知っていれば認証なくインターネット上から閲覧できてしまうことに注意**

## 手順

### 構築手順
1. 必要に応じてあらかじめ Azure OpenAI をデプロイしておく
   - このソースコードで Azure OpenAI もデプロイしたい場合は、下記手順から Azure OpenAI に関連する編集を行わない
   - GPT-4 Turbo with Vision を使わない場合は下記手順から GPT-4 Turbo with Vision に関連する編集を行わない
2. `azd auth login`
3. `azd init -t azure-search-openai-demo`
   - GitHub からダウンロード済みの場合は `azd init`
   - > Enter a new environment name: {envname}
4. /.azure/{envname}/.env に追加
   ```
   AZURE_LOCATION="japaneast"
   AZURE_OPENAI_CHATGPT_DEPLOYMENT="gpt-4-turbo"
   AZURE_OPENAI_CHATGPT_MODEL="gpt-4"
   AZURE_OPENAI_EMB_DEPLOYMENT="embedding"
   AZURE_OPENAI_EMB_MODEL_NAME="text-embedding-ada-002"
   AZURE_OPENAI_GPT4V_DEPLOYMENT="gpt-4v"
   AZURE_OPENAI_GPT4V_MODEL="gpt-4"
   AZURE_OPENAI_RESOURCE_GROUP="AOAI"
   AZURE_OPENAI_SERVICE="KoreAI-WestUS"
   AZURE_RESOURCE_GROUP="CogSearch"
   AZURE_SEARCH_ANALYZER_NAME="ja.microsoft"
   AZURE_SEARCH_INDEX="itconcierge"
   AZURE_SEARCH_QUERY_LANGUAGE="ja-JP"
   AZURE_SEARCH_QUERY_SPELLER="none"
   AZURE_STORAGE_CONTAINER="itconcierge"
   USE_GPT4V="true"
   ```
5. /infra/main.parameters.json に追加
   ```
   "storageContainerName": {
     "value": "${AZURE_STORAGE_CONTAINER}"
   },
   "chatGptModelName": {
     "value": "${AZURE_OPENAI_CHATGPT_MODEL}"
   },
   "gpt4vDeploymentName": {
     "value": "${AZURE_OPENAI_GPT4V_DEPLOYMENT}"
   },
   ```
6. /infra/main.bicep を修正
   | 変更前 | 変更後 |
   | --- | --- |
   | param storageContainerName string = 'content' | param storageContainerName string // Set in main.parameters.json |
   | @description('Location for the OpenAI resource group')<br>...<br>param openAiResourceGroupLocation string | 削除 |
   | param computerVisionResourceGroupLocation string = 'eastus' | param computerVisionResourceGroupLocation string = 'westus' |
   | AZURE_OPENAI_SERVICE: openAiHost == 'azure' ? openAi.outputs.name : '' |　AZURE_OPENAI_SERVICE: openAiHost == 'azure' ? openAiServiceName : '' |
   | module openAi 'core/ai/cognitiveservices.bicep' = {...} | 削除 |
   | module openAiRoleUser 'core/security/role.bicep' = if (openAiHost == 'azure') {...} | 削除 |
   | module openAiRoleBackend 'core/security/role.bicep' = if (openAiHost == 'azure') {...} | 削除 |
   | output AZURE_OPENAI_SERVICE string = (openAiHost == 'azure') ? openAi.outputs.name : '' | output AZURE_OPENAI_SERVICE string = (openAiHost == 'azure') ? openAiServiceName : '' |
7. /data にデータを配置
8. `azd up`
   - > Select an Azure Subscription to use: {サブスクリプション}

### データ反映手順
(ファイル名).md5 があり、かつ該当ファイルに更新がない場合は対象にならない  
**(ファイル名).md5 を削除して追加手順を実行すると重複するので注意**

#### データ追加
1. /.azure/{envname}/.env を編集
   ```
   AZURE_SEARCH_INDEX="hrconcierge"
   AZURE_STORAGE_CONTAINER="hrconcierge"
   ```
2. /data にデータを配置
3. (Win) `.\scripts\prepdocs.ps1`<br>
   (Linux/Mac) `/scripts/prepdocs.sh`

#### データ削除(一部)
1. prepdocs のラッパーを編集
   - (Win) /scripts/prepdocs.ps1
      | 変更前 | 変更後 |
      | --- | --- |
      | \$dataArg = "\`"$cwd/data/*\`"" | \$dataArg = "\`"$cwd/data/(ファイル名)\`"" |
      | $argumentList = "./scripts/prepdocs.py $dataArg | $argumentList = "./scripts/prepdocs.py $dataArg --remove |
   - (Linux/Mac) /scripts/prepdocs.sh
      | 変更前 | 変更後 |
      | --- | --- |
      | './data/*' | './data/(ファイル名)' --remove |
2. (Win) `.\scripts\prepdocs.ps1`<br>
   (Linux/Mac) `/scripts/prepdocs.sh`

#### データ削除(全部)
1. prepdocs のラッパーを編集
   - (Win) /scripts/prepdocs.ps1
      | 変更前 | 変更後 |
      | --- | --- |
      | $argumentList = "./scripts/prepdocs.py $dataArg | $argumentList = "./scripts/prepdocs.py $dataArg --removeall |
   - (Linux/Mac) /scripts/prepdocs.sh
      | 変更前 | 変更後 |
      | --- | --- |
      | './data/*' | './data/*' --removeall |
2. (Win) `.\scripts\prepdocs.ps1`<br>
   (Linux/Mac) `/scripts/prepdocs.sh`

### 削除手順
1. `azd down --purge`

(参考) 論理的削除(Soft delete)対象リソース
- Azure OpenAI
- Computer Vision
- Document intelligence
- キー コンテナー