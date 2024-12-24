# [プレビュー] AOAI を使用したサンプルチャットアプリ

このリポジトリには、Azure OpenAI を統合したシンプルなチャット Web アプリのサンプルコードが含まれています。一部のアプリ機能はプレビュー API を使用しています。

## 前提条件
- 既存の Azure OpenAI リソースおよびチャットモデルのデプロイメント（例: `gpt-35-turbo-16k`、`gpt-4`）
- Azure OpenAI をデータに使用する場合、以下のいずれかのデータソースが必要です:
  - Azure AI Search インデックス
  - Azure CosmosDB Mongo vCore ベクターインデックス
  - Elasticsearch インデックス（プレビュー）
  - Pinecone インデックス（プライベートプレビュー）
  - Azure SQL Server（プライベートプレビュー）
  - Mongo DB（プレビュー）

## アプリの構成

### ローカル開発用の .env ファイルを作成

以下の[アプリ設定](#アプリ設定)セクションに従って、ローカル開発用の .env ファイルを作成してください。このファイルは Azure App Service にデプロイされた Web アプリの設定を入力するための参照として使用できます。

### Azure App Service アプリ設定用の JSON ファイルを作成

.env ファイルを作成した後、以下のいずれかのコマンドを使用して、Azure App Service で認識される環境設定の JSON 表現を作成します。

#### PowerShell
```powershell
Get-Content .env | ForEach-Object {
    if ($_ -match "(?<name>[A-Z_]+)=(?<value>.*)") {
        [PSCustomObject]@{
            name = $matches["name"]
            value = $matches["value"]
            slotSetting = $false
        }
    }
} | ConvertTo-Json | Out-File -FilePath env.json
```

#### Bash
```bash
cat .env | jq -R '. | capture("(?<name>[A-Z_]+)=(?<value>.*)")' | jq -s '.[].slotSetting=false' > env.json
```

## アプリのデプロイ

### Azure Developer CLI を使用してデプロイ
詳細な手順については、[README_azd.md](./README_azd.md) を参照してください。

### ワンクリック Azure デプロイ
[![Azure にデプロイ](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmicrosoft%2Fsample-app-aoai-chatGPT%2Fmain%2Finfrastructure%2Fdeployment.json)

"Azure にデプロイ" ボタンをクリックし、Azure ポータルで [環境変数](#environment-variables) セクションに記載されている設定を構成してください。

### ローカルマシンからデプロイ

1. 以下の[アプリ設定](#アプリ設定)セクションに従い、使用ケースに適した変数で .env ファイルを作成してください。

2. `start.cmd` を使用してアプリを起動します。これにより、フロントエンドがビルドされ、バックエンド依存関係がインストールされ、アプリが起動します。または、`.vscode/launch.json` の VSCode デバッグ構成を使用してバックエンドをデバッグモードで実行します。

3. ローカルで実行中のアプリは http://127.0.0.1:50505 で確認できます。

### Azure CLI を使用してデプロイ

#### Azure App Service の作成
**注意**: コードを変更した場合は、デプロイ前に `start.cmd` または `start.sh` を使用してアプリコードをビルドしてください。フロントエンドフォルダ内のファイルを更新した場合は、デプロイ前に `static` フォルダに更新が反映されていることを確認してください。

Azure CLI を使用してローカルマシンからアプリをデプロイできます。バージョン 2.48.1 以降を使用してください。

アプリを初めてデプロイする場合、[az webapp up](https://learn.microsoft.com/en-us/cli/azure/webapp?view=azure-cli-latest#az-webapp-up) を使用できます。以下のコマンドをリポジトリのルートフォルダから実行してください。
```bash
az webapp up --runtime PYTHON:3.11 --sku B1 --name <new-app-name> --resource-group <resource-group-name> --location <azure-region> --subscription <subscription-name>
```

Azure CLI バージョン 2.62 以上を使用している場合、以下のフラグを追加することを検討してください。
```bash
--track-status False
```
このフラグは、起動エラーによるコマンドの失敗を防ぐためのものです。起動エラーは、次のセクションで説明する[アプリ構成の更新](#update-app-configuration)手順に従って解決してください。

#### アプリ構成の更新
Azure App Service を作成した後、アプリケーションを適切に起動するための構成を更新します。

1. アプリの起動コマンドを設定します。
```bash
az webapp config set --startup-file "python3 -m gunicorn app:app" --name <new-app-name>
```
2. ローカルコードデプロイを許可するために `WEBSITE_WEBDEPLOY_USE_SCM=false` を設定します。
```bash
az webapp config appsettings set -g <resource-group-name> -n <existing-app-name> --settings WEBSITE_WEBDEPLOY_USE_SCM=false
```
3. ローカル .env ファイルの設定を一括で適用します。
```bash
az webapp config appsettings set -g <resource-group-name> -n <existing-app-name> --settings "@env.json"
```

#### 既存のアプリの更新
Azure ポータルでアプリサービスリソースを表示してランタイムスタックを確認します。ランタイムが "Python - 3.10" の場合は `PYTHON:3.10` を、"Python - 3.11" の場合は `PYTHON:3.11` を使用します。

同様に SKU も確認し、適切な値を使用してください（例: "Basic (B1)" の場合は `B1`）。

次に、以下のコマンドを使用してローカルコードを既存のアプリにデプロイします。

1. `az webapp up --runtime <runtime-stack> --sku <sku> --name <existing-app-name> --resource-group <resource-group-name>`
1. `az webapp config set --startup-file "python3 -m gunicorn app:app" --name <existing-app-name>`

アプリ名とリソースグループ名が、以前デプロイされたアプリと正確に一致していることを確認してください。

デプロイには数分かかります。完了後、`{app-name}.azurewebsites.net` にアクセスしてアプリを確認できます。

## 認証

### ID プロバイダーの追加
デプロイ後、アプリに認証を提供するために ID プロバイダーを追加する必要があります。詳細については、[このチュートリアル](https://learn.microsoft.com/en-us/azure/app-service/scenario-secure-app-authentication-app-service)を参照してください。

ID プロバイダーを追加しない場合、アプリのチャット機能は、リソースやデータへの不正アクセスを防ぐためにブロックされます。

この制限を解除するには、環境変数に `AUTH_ENABLED=False` を追加できます。これにより認証が無効になり、誰でもアプリのチャット機能にアクセスできるようになります。  
**ただし、これは本番アプリには推奨されません。**

さらにアクセス制御を追加するには、`frontend/src/pages/chat/Chat.tsx` 内の `getUserInfoList` ロジックを更新してください。


### Using Microsoft Entra ID

To enable Microsoft Entra ID for intra-service authentication:

1. Enable managed identity on Azure OpenAI
2. Configure AI search to allow access from Azure OpenAI
   1. Enable Role Based Access control on the used AI search instance [(see documentation)](https://learn.microsoft.com/en-us/azure/search/search-security-enable-roles)
   2. Assign `Search Index Data Reader` and `Search Service Contributor` to the identity of the Azure OpenAI instance
3. Do not configure `AZURE_SEARCH_KEY` and `AZURE_OPENAI_KEY` to use Entra ID authentication.
4. Configure the webapp identity
   1. Enable managed identity in the app service that hosts the webapp
   2. Go to the Azure OpenAI instance and assign the role `Cognitive Services OpenAI User` to the identity of the webapp

Note: RBAC assignments can take a few minutes before becoming effective.

## アプリ設定

### アプリ設定

#### 基本的なチャット体験
1. `.env.sample` をコピーして、新しいファイル `.env` を作成し、以下の表に記載されている設定に従って設定を構成してください。

    | アプリ設定 | 必須か？ | デフォルト値 | 説明 |
    | --- | --- | --- | ------------- |
    |AZURE_OPENAI_RESOURCE| `AZURE_OPENAI_ENDPOINT` が設定されていない場合のみ必須 || Azure OpenAI リソースの名前（AZURE_OPENAI_RESOURCE または AZURE_OPENAI_ENDPOINT のいずれかが必要）|
    |AZURE_OPENAI_ENDPOINT| `AZURE_OPENAI_RESOURCE` が設定されていない場合のみ必須 || Azure OpenAI リソースのエンドポイント（AZURE_OPENAI_RESOURCE または AZURE_OPENAI_ENDPOINT のいずれかが必要）|
    |AZURE_OPENAI_MODEL|必須||モデルデプロイメントの名前|
    |AZURE_OPENAI_KEY|Microsoft Entra ID を使用する場合は任意 -- ID ベースの認証に必要なリソース設定についてはドキュメントを参照してください。||Azure OpenAI リソースの API キーのいずれか|
    |AZURE_OPENAI_TEMPERATURE|いいえ|0|サンプリング温度を 0 から 2 の間で設定。0.8 のような高い値は出力をよりランダムにし、0.2 のような低い値はより焦点が絞られ決定論的になります。データを使用する場合は 0 を推奨。|
    |AZURE_OPENAI_TOP_P|いいえ|1.0|サンプリング温度の代わりに「核サンプリング」と呼ばれる方法で、top_p 確率質量のトークンの結果をモデルが考慮します。データを使用する場合は 1.0 に設定することを推奨。|
    |AZURE_OPENAI_MAX_TOKENS|いいえ|1000|生成される回答の最大トークン数。|
    |AZURE_OPENAI_STOP_SEQUENCE|いいえ||API がトークンの生成を停止する最大 4 つのシーケンス。これらを "|" で結合した文字列として表現（例: `"stop1|stop2|stop3"`）。|
    |AZURE_OPENAI_SYSTEM_MESSAGE|いいえ|You are an AI assistant that helps people find information.|モデルが使用する役割とトーンの簡潔な説明。|
    |AZURE_OPENAI_STREAM|いいえ|True|レスポンスでストリーミングを使用するかどうかを設定します。注意: これを true に設定するとプロンプトフローは使用できなくなります。|
    |AZURE_OPENAI_EMBEDDING_NAME|Azure OpenAI 埋め込みモデルを使用してベクトル検索を行う場合のみ必須||ベクトル検索を使用する場合の埋め込みモデルデプロイメントの名前。|

    これらのパラメータの詳細については、[ドキュメント](https://learn.microsoft.com/en-us/azure/cognitive-services/openai/reference#example-response-2)を参照してください。



#### データとのチャット

[Azure OpenAI を使用したデータとのチャットに関する詳細情報](https://learn.microsoft.com/en-us/azure/cognitive-services/openai/concepts/use-your-data)

#### Azure Cognitive Search を使用したデータとのチャット

1. 上記の [基本的なチャット体験](#basic-chat-experience) で説明したように、`AZURE_OPENAI_*` 環境変数を更新してください。

2. データに接続するには、使用する Azure Cognitive Search インデックスを指定する必要があります。 [このインデックスを自分で作成する](https://learn.microsoft.com/en-us/azure/search/search-get-started-portal)か、[Azure AI Studio](https://oai.azure.com/portal/chat) を使用してインデックスを作成してください。

3. 以下の表に記載されたようにデータソースの設定を構成してください。

    | アプリ設定 | 必須か？ | デフォルト値 | 説明 |
    | --- | --- | --- | ------------- |
    |DATASOURCE_TYPE|はい||`AzureCognitiveSearch` に設定する必要があります。|
    |AZURE_SEARCH_SERVICE|はい||Azure AI Search リソースの名前。|
    |AZURE_SEARCH_INDEX|はい||Azure AI Search インデックスの名前。|
    |AZURE_SEARCH_KEY|Microsoft Entra ID を使用する場合は任意 -- ID ベースの認証に必要なリソース設定についてはドキュメントを参照してください。||Azure AI Search リソースの **管理キー**。|
    |AZURE_SEARCH_USE_SEMANTIC_SEARCH|いいえ|False|セマンティック検索を使用するかどうか。|
    |AZURE_SEARCH_QUERY_TYPE|いいえ|simple|クエリの種類: `simple`、`semantic`、`vector`、`vectorSimpleHybrid`、`vectorSemanticHybrid`。`AZURE_SEARCH_USE_SEMANTIC_SEARCH` より優先されます。|
    |AZURE_SEARCH_SEMANTIC_SEARCH_CONFIG|いいえ||セマンティック検索を使用する場合の設定名。|
    |AZURE_SEARCH_TOP_K|いいえ|5|検索インデックスをクエリする際に取得するドキュメントの数。|
    |AZURE_SEARCH_ENABLE_IN_DOMAIN|いいえ|True|クエリをデータに関連するものに限定します。|
    |AZURE_SEARCH_STRICTNESS|いいえ|3|モデルが応答をデータに限定する厳密性を指定する 1 から 5 の整数。|
    |AZURE_SEARCH_CONTENT_COLUMNS|いいえ||検索インデックス内の、ボットの応答を構成するために使用されるドキュメントのテキストコンテンツを含むフィールドのリスト。`|` で結合（例: `"product_description|product_manual"`）。|
    |AZURE_SEARCH_FILENAME_COLUMN|いいえ||UI に表示するデータソースの一意の識別子を提供する検索インデックスのフィールド。|
    |AZURE_SEARCH_TITLE_COLUMN|いいえ||UI に表示するデータコンテンツの関連タイトルまたはヘッダーを提供する検索インデックスのフィールド。|
    |AZURE_SEARCH_URL_COLUMN|いいえ||ドキュメントの URL を含む検索インデックスのフィールド（例: Azure Blob Storage URI）。現在は使用されていません。|
    |AZURE_SEARCH_VECTOR_COLUMNS|いいえ||ボットの応答を構成するために使用されるドキュメントのベクトル埋め込みを含む検索インデックスのフィールドのリスト。`|` で結合（例: `"product_description|product_manual"`）。|
    |AZURE_SEARCH_PERMITTED_GROUPS_COLUMN|いいえ||ドキュメントレベルのアクセス制御を決定する AAD グループ ID を含む Azure AI Search インデックスのフィールド。|

    独自のデータとベクトルインデックスを使用する場合、以下の設定をアプリに構成してください：
    - `AZURE_SEARCH_QUERY_TYPE`: `vector`、`vectorSimpleHybrid`、または `vectorSemanticHybrid` に設定。
    - `AZURE_OPENAI_EMBEDDING_NAME`: Azure OpenAI リソース上の Ada（`text-embedding-ada-002`）モデルデプロイメントの名前。
    - `AZURE_SEARCH_VECTOR_COLUMNS`: 検索に使用するインデックス内のベクトル列。`|` で結合（例: `contentVector|titleVector`）。


#### Azure Cosmos DB を使用したデータとのチャット

1. 上記の [基本的なチャット体験](#basic-chat-experience) で説明したように、`AZURE_OPENAI_*` 環境変数を更新してください。

2. データに接続するには、Azure Cosmos DB データベースの設定を指定する必要があります。[Azure Cosmos DB リソースの作成方法についてはこちら](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/quickstart-portal)をご覧ください。

3. 以下の表に記載されたようにデータソースの設定を構成してください。

    | アプリ設定 | 必須か？ | デフォルト値 | 説明 |
    | --- | --- | --- | ------------- |
    |DATASOURCE_TYPE|はい||`AzureCosmosDB` に設定する必要があります。|
    |AZURE_COSMOSDB_MONGO_VCORE_CONNECTION_STRING|はい||Azure Cosmos DB インスタンスに接続するために使用される接続文字列。|
    |AZURE_COSMOSDB_MONGO_VCORE_INDEX|はい||Azure Cosmos DB ベクトルインデックスの名前。|
    |AZURE_COSMOSDB_MONGO_VCORE_DATABASE|はい||Azure Cosmos DB データベースの名前。|
    |AZURE_COSMOSDB_MONGO_VCORE_CONTAINER|はい||Azure Cosmos DB コンテナーの名前。|
    |AZURE_COSMOSDB_MONGO_VCORE_TOP_K|いいえ|5|検索インデックスをクエリする際に取得するドキュメントの数。|
    |AZURE_COSMOSDB_MONGO_VCORE_ENABLE_IN_DOMAIN|いいえ|True|クエリをデータに関連するものに限定します。|
    |AZURE_COSMOSDB_MONGO_VCORE_STRICTNESS|いいえ|3|モデルが応答をデータに限定する厳密性を指定する 1 から 5 の整数。|
    |AZURE_COSMOSDB_MONGO_VCORE_CONTENT_COLUMNS|いいえ||検索インデックス内の、ボットの応答を構成するために使用されるドキュメントのテキストコンテンツを含むフィールドのリスト。`|` で結合（例: `"product_description|product_manual"`）。|
    |AZURE_COSMOSDB_MONGO_VCORE_FILENAME_COLUMN|いいえ||UI に表示するデータソースの一意の識別子を提供する検索インデックスのフィールド。|
    |AZURE_COSMOSDB_MONGO_VCORE_TITLE_COLUMN|いいえ||UI に表示するデータコンテンツの関連タイトルまたはヘッダーを提供する検索インデックスのフィールド。|
    |AZURE_COSMOSDB_MONGO_VCORE_URL_COLUMN|いいえ||ドキュメントの URL を含む検索インデックスのフィールド（例: Azure Blob Storage URI）。現在は使用されていません。|
    |AZURE_COSMOSDB_MONGO_VCORE_VECTOR_COLUMNS|いいえ||ボットの応答を構成するために使用されるドキュメントのベクトル埋め込みを含む検索インデックスのフィールドのリスト。`|` で結合（例: `"product_description|product_manual"`）。|

    Azure Cosmos DB はデフォルトでベクトル検索を使用するため、以下の設定をアプリに構成してください：
    - `AZURE_OPENAI_EMBEDDING_NAME`: Azure OpenAI リソース上の Ada（`text-embedding-ada-002`）モデルデプロイメントの名前。
    - `AZURE_COSMOSDB_MONGO_VCORE_VECTOR_COLUMNS`: 検索に使用するインデックス内のベクトル列。`|` で結合（例: `contentVector|titleVector`）。


#### Chat with your data using Elasticsearch (Preview)

1. Update the `AZURE_OPENAI_*` environment variables as described in the [basic chat experience](#basic-chat-experience) above. 

2. To connect to your data, you need to specify an Elasticsearch cluster configuration. Learn more about [Elasticsearch](https://www.elastic.co/).

3. Configure data source settings as described in the table below.

    | App Setting | Required? | Default Value | Note |
    | --- | --- | --- | ------------- |
    |DATASOURCE_TYPE|Yes||Must be set to `Elasticsearch`|
    |ELASTICSEARCH_ENDPOINT|Yes||The base URL of your Elasticsearch cluster API|
    |ELASTICSEARCH_ENCODED_API_KEY|Yes||The encoded API key for your user identity on your Elasticsearch cluster|
    |ELASTICSEARCH_INDEX|Yes||The name of your Elasticsearch index|
    |ELASTICSEARCH_QUERY_TYPE|No|simple|Can be one of `simple` or `vector`|
    |ELASTICSEARCH_EMBEDDING_MODEL_ID|Only if using vector search with an Elasticsearch embedding model||The name of the embedding model deployed to your Elasticsearch cluster which was used to produce embeddings for your index|
    |ELASTICSEARCH_TOP_K|No|5|The number of documents to retrieve when querying your search index.|
    |ELASTICSEARCH_ENABLE_IN_DOMAIN|No|True|Limits responses to only queries relating to your data.|
    |ELASTICSEARCH_STRICTNESS|No|3|Integer from 1 to 5 specifying the strictness for the model limiting responses to your data.|
    |ELASTICSEARCH_CONTENT_COLUMNS|No||List of fields in your search index that contains the text content of your documents to use when formulating a bot response. Represent these as a string joined with "|", e.g. `"product_description|product_manual"`|
    |ELASTICSEARCH_FILENAME_COLUMN|No|| Field from your search index that gives a unique identifier of the source of your data to display in the UI.|
    |ELASTICSEARCH_TITLE_COLUMN|No||Field from your search index that gives a relevant title or header for your data content to display in the UI.|
    |ELASTICSEARCH_URL_COLUMN|No||Field from your search index that contains a URL for the document, e.g. an Azure Blob Storage URI. This value is not currently used.|
    |ELASTICSEARCH_VECTOR_COLUMNS|No||List of fields in your search index that contain vector embeddings of your documents to use when formulating a bot response. Represent these as a string joined with "|", e.g. `"product_description|product_manual"`|

    To use vector search with Elasticsearch, there are two options:

    1. To use Azure OpenAI embeddings, ensure that your index contains Azure OpenAI embeddings, and that the following variables are set:
    - `AZURE_OPENAI_EMBEDDING_NAME`: the name of your Ada (text-embedding-ada-002) model deployment on your Azure OpenAI resource, which was also used to create the embeddings in your index.
    - `ELASTICSEARCH_VECTOR_COLUMNS`: the vector columns in your index to use when searching. Join them with `|` like `contentVector|titleVector`.

    2. Use Elasticsearch embeddings, ensure that your index contains embeddings produced from a trained model on your Elasticsearch cluster, and that the following variables are set:
    - `ELASTICSEARCH_EMBEDDING_MODEL_ID`: the ID of the trained model used to produce embeddings on your index.
    - `ELASTICSEARCH_VECTOR_COLUMNS`: the vector columns in your index to use when searching. Join them with `|` like `contentVector|titleVector`.

#### Pinecone を使用したデータとのチャット（プライベートプレビュー）

1. 上記の [基本的なチャット体験](#basic-chat-experience) で説明したように、`AZURE_OPENAI_*` 環境変数を更新してください。

2. データに接続するには、Pinecone ベクトルデータベースの設定を指定する必要があります。[Pinecone についてはこちら](https://www.pinecone.io/)をご覧ください。

3. 以下の表に記載されたようにデータソースの設定を構成してください。

    | アプリ設定 | 必須か？ | デフォルト値 | 説明 |
    | --- | --- | --- | ------------- |
    |DATASOURCE_TYPE|はい||`Pinecone` に設定する必要があります。|
    |PINECONE_ENVIRONMENT|はい||Pinecone 環境の名前。|
    |PINECONE_INDEX_NAME|はい||Pinecone インデックスの名前。|
    |PINECONE_API_KEY|はい||Pinecone インスタンスに接続するために使用される API キー。|
    |PINECONE_TOP_K|いいえ|5|検索インデックスをクエリする際に取得するドキュメントの数。|
    |PINECONE_ENABLE_IN_DOMAIN|いいえ|True|クエリをデータに関連するものに限定します。|
    |PINECONE_STRICTNESS|いいえ|3|モデルが応答をデータに限定する厳密性を指定する 1 から 5 の整数。|
    |PINECONE_CONTENT_COLUMNS|いいえ||検索インデックス内の、ボットの応答を構成するために使用されるドキュメントのテキストコンテンツを含むフィールドのリスト。`|` で結合（例: `"product_description|product_manual"`）。|
    |PINECONE_FILENAME_COLUMN|いいえ||UI に表示するデータソースの一意の識別子を提供する検索インデックスのフィールド。|
    |PINECONE_TITLE_COLUMN|いいえ||UI に表示するデータコンテンツの関連タイトルまたはヘッダーを提供する検索インデックスのフィールド。|
    |PINECONE_URL_COLUMN|いいえ||ドキュメントの URL を含む検索インデックスのフィールド（例: Azure Blob Storage URI）。現在は使用されていません。|
    |PINECONE_VECTOR_COLUMNS|いいえ||ボットの応答を構成するために使用されるドキュメントのベクトル埋め込みを含む検索インデックスのフィールドのリスト。`|` で結合（例: `"product_description|product_manual"`）。|

    Pinecone はデフォルトでベクトル検索を使用するため、以下の設定をアプリに構成してください：
    - `AZURE_OPENAI_EMBEDDING_NAME`: Azure OpenAI リソース上の Ada（`text-embedding-ada-002`）モデルデプロイメントの名前。
    - `PINECONE_VECTOR_COLUMNS`: 検索に使用するインデックス内のベクトル列。`|` で結合（例: `contentVector|titleVector`）。


#### Chat with your data using Mongo DB (Private Preview)

1. Update the `AZURE_OPENAI_*` environment variables as described in the [basic chat experience](#basic-chat-experience) above. 

2. To connect to your data, you need to specify an Mongo DB database configuration.  Learn more about [MongoDB](https://www.mongodb.com/).

3. Configure data source settings as described in the table below.

    | App Setting | Required? | Default Value | Note |
    | --- | --- | --- | ------------- |
    |DATASOURCE_TYPE|Yes||Must be set to `MongoDB`|
    |MONGODB_CONNECTION_STRING|Yes||The connection string used to connect to your Mongo DB instance|
    |MONGODB_VECTOR_INDEX|Yes||The name of your Mongo DB vector index|
    |MONGODB_DATABASE_NAME|Yes||The name of your Mongo DB database|
    |MONGODB_CONTAINER_NAME|Yes||The name of your Mongo DB container|
    |MONGODB_TOP_K|No|5|The number of documents to retrieve when querying your search index.|
    |MONGODB_ENABLE_IN_DOMAIN|No|True|Limits responses to only queries relating to your data.|
    |MONGODB_STRICTNESS|No|3|Integer from 1 to 5 specifying the strictness for the model limiting responses to your data.|
    |MONGODB_CONTENT_COLUMNS|No||List of fields in your search index that contains the text content of your documents to use when formulating a bot response. Represent these as a string joined with "|", e.g. `"product_description|product_manual"`|
    |MONGODB_FILENAME_COLUMN|No|| Field from your search index that gives a unique identifier of the source of your data to display in the UI.|
    |MONGODB_TITLE_COLUMN|No||Field from your search index that gives a relevant title or header for your data content to display in the UI.|
    |MONGODB_URL_COLUMN|No||Field from your search index that contains a URL for the document, e.g. an Azure Blob Storage URI. This value is not currently used.|
    |MONGODB_VECTOR_COLUMNS|No||List of fields in your search index that contain vector embeddings of your documents to use when formulating a bot response. Represent these as a string joined with "|", e.g. `"product_description|product_manual"`|

    MongoDB uses vector search by default, so ensure these settings are configured on your app:
    - `AZURE_OPENAI_EMBEDDING_NAME`: the name of your Ada (text-embedding-ada-002) model deployment on your Azure OpenAI resource.
    - `MONGODB_VECTOR_COLUMNS`: the vector columns in your index to use when searching. Join them with `|` like `contentVector|titleVector`.
    
#### Chat with your data using Azure SQL Server (Private Preview)

1. Update the `AZURE_OPENAI_*` environment variables as described in the [basic chat experience](#basic-chat-experience) above. 

2. To enable Azure SQL Server, you will need to set up Azure SQL Server resources.  Refer to this [instruction guide](https://learn.microsoft.com/en-us/azure/azure-sql/database/single-database-create-quickstart) to create an Azure SQL database. 

3. Configure data source settings as described in the table below.

    | App Setting | Required? | Default Value | Note |
    | --- | --- | --- | ------------- |
    |DATASOURCE_TYPE|Yes||Must be set to `AzureSqlServer`|
    |AZURE_SQL_SERVER_CONNECTION_STRING|Yes||The connection string to use to connect to your Azure SQL Server instance|
    |AZURE_SQL_SERVER_TABLE_SCHEMA|Yes||The table schema for your Azure SQL Server table.  Must be surrounded by double quotes (`"`).|
    |AZURE_SQL_SERVER_PORT||Not publicly available at this time.|The port to use to connect to your Azure SQL Server instance.|
    |AZURE_SQL_SERVER_DATABASE_NAME||Not publicly available at this time.|
    |AZURE_SQL_SERVER_DATABASE_SERVER||Not publicly available at this time.|

#### Promptflow を使用したデータとのチャット

以下の表を使用して設定を構成します。

| アプリ設定 | 必須か？ | デフォルト値 | 説明 |
| --- | --- | --- | ------------- |
|USE_PROMPTFLOW|いいえ|False|既存の Promptflow デプロイメントエンドポイントを使用します。`True` に設定すると、`PROMPTFLOW_ENDPOINT` および `PROMPTFLOW_API_KEY` も設定する必要があります。|
|PROMPTFLOW_ENDPOINT|`USE_PROMPTFLOW` が True の場合のみ||デプロイされた Promptflow エンドポイントの URL（例: https://pf-deployment-name.region.inference.ml.azure.com/score）。|
|PROMPTFLOW_API_KEY|`USE_PROMPTFLOW` が True の場合のみ||デプロイされた Promptflow エンドポイントの認証キー。注意：キーベースの認証のみがサポートされています。|
|PROMPTFLOW_RESPONSE_TIMEOUT|いいえ|120|Promptflow エンドポイントが応答するまでのタイムアウト値（秒単位）。|
|PROMPTFLOW_REQUEST_FIELD_NAME|いいえ|query|Promptflow リクエストを構築する際のデフォルトのフィールド名。注意：チャット履歴は自動で構築されますが、APIが他の必須のフィールドを期待する場合は、`promptflow_request` 関数のリクエストパラメータを変更する必要があります。|
|PROMPTFLOW_RESPONSE_FIELD_NAME|いいえ|reply|Promptflow からの応答を処理する際のデフォルトのフィールド名。|
|PROMPTFLOW_CITATIONS_FIELD_NAME|いいえ|documents|Promptflow からの引用出力を処理する際のデフォルトのフィールド名。|

#### チャット履歴の有効化

1. 上記の [基本的なチャット体験](#basic-chat-experience) で説明したように、`AZURE_OPENAI_*` 環境変数を更新します。

2. 必要に応じて、データとチャットするための追加の構成（前のセクションで説明）を追加します。

3. チャット履歴を有効にするには、Cosmos DB リソースの設定が必要です。`infrastructure` フォルダ内の ARM テンプレートを使用して、アプリサービスと Cosmos DB をデプロイし、データベースとコンテナーを構成します。

4. 以下の表に基づいてデータソース設定を構成します。

| アプリ設定 | 必須か？ | デフォルト値 | 説明 |
| --- | --- | --- | ------------- |
|AZURE_COSMOSDB_ACCOUNT|チャット履歴を使用する場合のみ||チャット履歴を保存するために使用する Azure Cosmos DB アカウントの名前。|
|AZURE_COSMOSDB_DATABASE|チャット履歴を使用する場合のみ||チャット履歴を保存するために使用する Azure Cosmos DB データベースの名前。|
|AZURE_COSMOSDB_CONVERSATIONS_CONTAINER|チャット履歴を使用する場合のみ||チャット履歴を保存するために使用する Azure Cosmos DB コンテナーの名前。|
|AZURE_COSMOSDB_ACCOUNT_KEY|チャット履歴を使用する場合のみ||チャット履歴を保存するために使用する Azure Cosmos DB アカウントのキー。|
|AZURE_COSMOSDB_ENABLE_FEEDBACK|いいえ|False|チャット履歴メッセージへのフィードバックを有効にするかどうか。|

#### 一般的なカスタマイズシナリオ (例: デフォルトのチャットロゴやヘッダーの更新)

インターフェースのタイトルやロゴなど、いくつかの要素を環境変数を使用して簡単にカスタマイズすることができます。

| アプリ設定 | 必須か？ | デフォルト値 | 説明 |
|---|---|---|---|
|UI_TITLE|いいえ|Contoso|チャットのタイトル（左上部）とページのタイトル（HTML）|
|UI_LOGO|いいえ||ロゴ（左上部）。デフォルトは Contoso ロゴです。ロゴ画像の URL を設定してカスタマイズします。|
|UI_CHAT_LOGO|いいえ||チャットウィンドウのロゴ。デフォルトは Contoso ロゴです。ロゴ画像の URL を設定してカスタマイズします。|
|UI_CHAT_TITLE|いいえ|Start chatting|チャットウィンドウのタイトル|
|UI_CHAT_DESCRIPTION|いいえ|This chatbot is configured to answer your questions|チャットウィンドウの説明|
|UI_FAVICON|いいえ||デフォルトは Contoso のファビコンです。ファビコンの URL を設定してカスタマイズします。|
|UI_SHOW_SHARE_BUTTON|いいえ|True|共有ボタン（右上部）を表示するかどうか。|
|UI_SHOW_CHAT_HISTORY_BUTTON|いいえ|True|チャット履歴ボタン（右上部）を表示するかどうか。|
|SANITIZE_ANSWER|いいえ|False|Azure OpenAI からの応答をサニタイズするかどうか。True に設定すると、応答からすべての HTML タグが削除されます。|

`UI_LOGO`、`UI_CHAT_LOGO`、`UI_FAVICON` に割り当てたカスタム画像は、プロジェクトの [public](https://github.com/microsoft/sample-app-aoai-chatGPT/tree/main/frontend/public) フォルダに追加する必要があります。Vite ビルドプロセスにより、これらのファイルが [static](https://github.com/microsoft/sample-app-aoai-chatGPT/tree/main/static) フォルダに自動でコピーされます。これらの環境変数を設定する際には、フロントエンドコードが見つけられるように、相対パス `static/<ファイル名>` を指定する必要があります。

このリポジトリをフォークして、UX やバックエンドのロジックをカスタマイズすることも可能です。ソース (`frontend/src`) を変更することで、チャットの表示や UI の一部を改良したり、`app.py` 内の設定をユーザーが試せるように公開することもできます。コード変更後は、`start.sh` または `start.cmd` を使用してフロントエンドを再ビルドする必要があります。

### スケーラビリティ
`gunicorn.conf.py` でスレッド数やワーカー数を構成できます。変更を加えた後は、上記のコマンドでアプリを再デプロイします。

### デプロイされたアプリのデバッグ
まず、アプリサービスリソースに「DEBUG」という環境変数を追加し、「true」に設定します。

次に、アプリサービスでログを有効にします。「監視」の下の「アプリサービスログ」に移動し、アプリログの種類を「ファイルシステム」に変更して保存します。

これで、「監視」の「ログストリーム」からアプリのログを確認できるようになります。

### 引用の表示の変更
引用パネルは、`frontend/src/pages/chat/Chat.tsx` の末尾で定義されています。Azure OpenAI On Your Data から返された引用には、`content`、`title`、`filepath`、場合によっては `url` が含まれています。これをカスタマイズして表示方法を調整できます。たとえば、タイトル要素は、`url` が BLOB URL でない場合にクリック可能なハイパーリンクになります。

```javascript
<h5 
    className={styles.citationPanelTitle} 
    tabIndex={0} 
    title={activeCitation.url && !activeCitation.url.includes("blob.core") ? activeCitation.url : activeCitation.title ?? ""} 
    onClick={() => onViewSource(activeCitation)}
>{activeCitation.title}</h5>

const onViewSource = (citation: Citation) => {
    if (citation.url && !citation.url.includes("blob.core")) {
        window.open(citation.url, "_blank");
    }
};
```

## ベストプラクティス

以下のベストプラクティスを念頭に置いておくことをお勧めします：

- ユーザーが設定を変更した場合は、チャットセッションをリセット（チャットをクリア）し、チャット履歴が失われることを通知します。
- 各設定がユーザーにどのような影響を与えるかを明確に伝えましょう。
- AOAI または ACS リソースの API キーを更新する際は、デプロイ済みのアプリすべてで新しいキーを使用するようにアプリ設定を更新することを忘れずに行いましょう。
- Azure OpenAI でのデータ利用時に、バグ修正や改善を確保するために `main` から頻繁に変更を取り込むことをお勧めします。

**Azure OpenAI API バージョンについての注意点**:
このリポジトリのアプリケーションコードは、Azure OpenAI における最新のプレビュー API バージョンに基づいたリクエストと応答の契約を実装しています。時間の経過と共に Azure OpenAI API が進化する中で、アプリケーションを最新の状態に保つためには、リポジトリの最新の API バージョン更新を自分のアプリケーションコードにマージし、ドキュメントに記載された手法で再デプロイする必要があります。


## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

When contributing to this repository, please help keep the codebase clean and maintainable by running 
the formatter and linter with `npm run format` this will run `npx eslint --fix` and `npx prettier --write` 
on the frontebnd codebase. 

If you are using VSCode, you can add the following settings to your `settings.json` to format and lint on save:

```json
{
    "editor.codeActionsOnSave": {
        "source.fixAll.eslint": "explicit"
    },
    "editor.formatOnSave": true,
    "prettier.requireConfig": true,
}
```

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft 
trademarks or logos is subject to and must follow 
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
