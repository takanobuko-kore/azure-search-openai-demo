# 1つのリソースで複数テナント用にデータソースを分けて構築する方法

## 背景
Azure AI Search リソースは up にしているだけで月5万程度の金額がかかってしまうため、1つのリソースで複数テナント用に提供したい

## 前提
下記手順は、原則 Japan East での構築を想定  
**ただし、`text-embedding-3` は Japan East で使用不可**  
※ 2024-05-03 現在  
[Azure OpenAI Service モデル#パブリック クラウド リージョン](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/concepts/models#public-cloud-regions-2)

**統合ベクター化はパブリック プレビュー段階**  
※ 2024-05-03 現在  
[Azure AI Search 内の統合データのチャンキングと埋め込み](https://learn.microsoft.com/ja-jp/azure/search/vector-search-integrated-vectorization)

データソースが論理的に分かれているため、検索の過程においてはあるテナントが違うテナントのリソースを検索してしまうことはない  
**ただし、データソースとして提示される URL は public のため、URL さえ知っていれば認証なくインターネット上から閲覧できてしまうことに注意**  
※ Azure ADと連携してログインしているユーザーに基づいてインデックス付きデータへのアクセスを制限できる(未検証)  
[Enabling optional features#Enabling login and document level access control](https://github.com/takanobuko-kore/azure-search-openai-demo/blob/main/docs/deploy_features.md#enabling-user-document-upload)

## 手順

### 構築手順
1. 必要に応じてあらかじめ Azure OpenAI をデプロイしておく
2. `azd auth login`
3. `azd init -t azure-search-openai-demo`
   - GitHub からダウンロード済みの場合は `azd env new`
   - > Enter a new environment name: {環境名}
4. /.azure/{env環境名name}/.env に追加
   ```
   AZURE_LOCATION="japaneast"
   AZURE_SEARCH_ANALYZER_NAME="ja.microsoft"
   AZURE_SEARCH_QUERY_LANGUAGE="ja-JP"
   AZURE_SEARCH_QUERY_SPELLER="none"
   AZURE_SEARCH_SEMANTIC_RANKER="standard"

   # 変数
   AZURE_ENV_NAME="{環境名}"
   AZURE_RESOURCE_GROUP="{リソース グループ名}"
   AZURE_SEARCH_INDEX="{検索サービス インデックス名}"
   AZURE_STORAGE_CONTAINER="{ストレージ アカウント コンテナー名}"

   # デプロイ済みの Azure OpenAI を使用する場合
   AZURE_OPENAI_RESOURCE_GROUP="{Azure OpenAI リソースがあるリソース グループ名}"
   AZURE_OPENAI_SERVICE="{Azure OpenAI リソース名}"
   AZURE_OPENAI_CHATGPT_MODEL="{モデル名}"
   AZURE_OPENAI_CHATGPT_DEPLOYMENT="{デプロイ名}"
   AZURE_OPENAI_EMB_DEPLOYMENT="{埋め込みモデル デプロイ名}"

   # コストを最小化する場合
   AZURE_APP_SERVICE_SKU="F1"
   AZURE_USE_APPLICATION_INSIGHTS="false"

   # 統合ベクター化を使用する場合
   USE_FEATURE_INT_VECTORIZATION="true"

   # text-embedding-3 を使用する場合
   AZURE_OPENAI_EMB_MODEL_NAME="text-embedding-3-large"
   AZURE_OPENAI_EMB_DEPLOYMENT_VERSION=1

   # GPT-4V を使用する場合
   USE_GPT4V="true"
   AZURE_COMPUTER_VISION_LOCATION="japaneast"
   ```
1. /infra/main.parameters.json に追加
   ```
   "storageContainerName": {
     "value": "${AZURE_STORAGE_CONTAINER}"
   },
   ```
2. /infra/main.bicep を修正
   | 変更前 | 変更後 |
   | --- | --- |
   | param storageContainerName string = 'content' | param storageContainerName string // Set in main.parameters.json |
   | @description('Location for the Document Intelligence resource group')<br>@allowed([ 'eastus', 'westus2', 'westeurope' ]) | @description('Location for the Document Intelligence resource group')<br>@allowed([ 'eastus', 'westus2', 'westeurope', 'japaneast' ]) |
3. /data にデータを配置
4. `azd up`
   - > Select an Azure Subscription to use: {サブスクリプション}

### データ反映手順
(ファイル名).md5 があり、かつ該当ファイルに更新がない場合は対象にならない  
**(ファイル名).md5 を削除して追加手順を実行すると重複するので注意**

#### データ / 新インデックス+コンテナー追加
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
1. `azd down`

(参考) 論理的削除(Soft delete)対象リソース
- Azure OpenAI
- Computer Vision
- Document intelligence
- キー コンテナー