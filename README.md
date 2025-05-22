# スマートFAQボット (bedrock-knowledge-base-qa-bot)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Python Version](https://img.shields.io/badge/python-3.11-blue.svg)](https://www.python.org/downloads/)
[![Framework: FastAPI](https://img.shields.io/badge/Framework-FastAPI-green.svg)](https://fastapi.tiangolo.com/)
[![AWS Services](https://img.shields.io/badge/AWS-Bedrock%2C%20Lambda%2C%20APIGateway%2C%20S3%2C%20OpenSearch%2C%20Cognito-orange.svg)](https://aws.amazon.com/)

**スマートFAQボット** は、Amazon Bedrockの強力なナレッジベースと生成AIモデルを活用し、FastAPIで構築され、AWS Lambda上で動作するインテリジェントな社内FAQ応答APIです。Amazon Cognitoによる認証機能を備え、セキュアに従業員の疑問を迅速に解決します。

## 概要 (Overview)

このAPIは、社内情報へのアクセスを容易にし、従業員の生産性向上に貢献することを目的としています。

**主な機能:**

1.  **AI搭載 FAQ API (`/faq/ask`):**
    *   社内規定、ITサポート手順、福利厚生情報など、S3に格納されたドキュメント群をナレッジソースとします。
    *   自然言語による質問に対し、Amazon Bedrockナレッジベース (Amazon OpenSearch Serverlessを利用) を検索し、選択した基盤モデル (例: Anthropic Claude 3 Sonnet) によって文脈に沿った分かりやすい回答を生成します。
2.  **サポートチケット起票支援 API (`/support/create-ticket`):** (※アレンジ例)
    *   FAQで解決しない問題について、ユーザーが自然言語で問題を説明すると、それを構造化し、社内のチケットシステム (例: Jira, Zendesk) へ起票するためのデータ形式に変換、または直接起票する支援を行います。

## 特徴 (Features)

*   **サーバーレスアーキテクチャ:** AWS Lambdaによるスケーラブルでコスト効率の高い運用。
*   **高速かつ効率的なAPI:** FastAPIによる非同期処理と高いパフォーマンス。
*   **高精度なFAQ検索:** Amazon Bedrockナレッジベース (Amazon OpenSearch Serverless, `amazon.titan-embed-text-v1`埋め込みモデル) によるセマンティック検索と、最新LLMによる自然な回答生成。
*   **セキュアなアクセス:** Amazon Cognitoによる認証・認可。
*   **OpenAPI準拠:** 自動生成されるSwagger UIとReDocによるAPIドキュメント。
*   **拡張性:** 新たなFAQソースや、シンプルな業務連携機能の追加が比較的容易。

## アーキテクチャ (Conceptual Architecture)

```mermaid
graph TD
    subgraph "ユーザー/クライアントシステム"
        Client[社内ポータル / チャットボット / etc.]
    end

    Client -- HTTPS (質問/業務指示) --> APIGW[Amazon API Gateway (HTTP API)];
    APIGW -- Cognito Authorizer --> Lambda[AWS Lambda (FastAPI)];

    subgraph "FastAPI Application on Lambda (スマートFAQボット)"
        Lambda -- "/faq/ask" --> FAQService[FAQ Service];
        FAQService -- Bedrock KB Query --> BedrockKB[Amazon Bedrock Knowledge Base];
        BedrockKB -- Search (OpenSearch Serverless) --> VectorStore[Amazon OpenSearch Serverless];
        BedrockKB -- Generate Answer (e.g., Claude 3 Sonnet) --> FAQService;
        FAQService -- Formatted Answer --> Lambda;

        Lambda -- "/support/create-ticket" --> SupportTicketService[Support Ticket Service];
        SupportTicketService -- Parse & Structure Request --> SupportTicketService;
        SupportTicketService -- API Call (e.g., to Jira) --> ExternalTicketSystem[社内チケットシステム API];
        ExternalTicketSystem -- Ticket Creation Result --> SupportTicketService;
        SupportTicketService -- Result --> Lambda;
    end

    S3Docs[社内ドキュメント (S3)] -- Ingestion --> BedrockKB;

    Lambda -- JSON Response --> APIGW;
    APIGW -- Response --> Client;

    style BedrockKB fill:#FF9900,stroke:#333,stroke-width:2px
    style S3Docs fill:#232F3E,stroke:#FF9900,stroke-width:2px,color:#fff
    style APIGW fill:#5A30B5,stroke:#333,stroke-width:2px,color:#fff
    style Lambda fill:#F9A825,stroke:#333,stroke-width:2px,color:#fff
    style VectorStore fill:#0073BB,stroke:#333,stroke-width:2px,color:#fff
```

## 使用技術 (Technology Stack)

*   **クラウドプラットフォーム:** AWS
*   **主要AWSサービス:**
    *   **Amazon Bedrock:** ナレッジベース、基盤モデル (FAQ回答生成用: 例 `anthropic.claude-3-sonnet-20240229-v1:0`、埋め込み用: `amazon.titan-embed-text-v1`)
    *   **AWS Lambda:** FastAPIアプリケーションホスティング
    *   **Amazon API Gateway (HTTP API):** APIエンドポイント公開、Cognitoオーソライザー連携
    *   **Amazon S3:** FAQドキュメントストレージ、ナレッジベースソース (例: `s3://your-faq-documents-bucket/`)
    *   **Amazon OpenSearch Serverless:** Bedrockナレッジベース用ベクトルストア
    *   **Amazon Cognito:** ユーザー認証・認可
    *   **Amazon CloudWatch:** ロギング、モニタリング
    *   **AWS IAM:** 権限管理
*   **プログラミング言語:** Python 3.11
*   **Webフレームワーク:** FastAPI
*   **主要ライブラリ (アレンジ例):**
    *   Boto3: AWS SDK for Python
    *   Uvicorn: ASGIサーバー (ローカル開発用)
    *   Pydantic (v2推奨): データバリデーションと設定管理
    *   Mangum: AWS LambdaでFastAPIアプリケーションを実行するためのアダプタ
    *   HTTPX: 高度なHTTPクライアント (外部API連携用、例: チケットシステム)
    *   Loguru (推奨): シンプルで強力なロギング
    *   `python-dotenv`: 環境変数管理 (ローカル開発用)
    *   `aws-lambda-powertools`: AWS Lambda開発のベストプラクティス (ロギング、トレーシング、メトリクスなど)
*   **CI/CD (推奨):** GitHub Actions, AWS CodePipeline (AWS SAMやServerless Frameworkと連携)

## セットアップと実行方法 (Setup & Usage)

### 1. 前提条件 (Prerequisites)

*   AWSアカウントと設定済みのAWS CLI (v2推奨)
*   Python 3.11 および Poetry (推奨) または pip
*   AWS SAM CLI (推奨、Lambdaデプロイ用) または Serverless Framework
*   Git

### 2. リポジトリのクローン

```bash
git clone https://github.com/[あなたのGitHubユーザー名]/bedrock-knowledge-base-qa-bot.git
cd bedrock-knowledge-base-qa-bot
```

### 3. 依存関係のインストール (Poetryを使用する場合)

```bash
poetry install
```
*(pip を使用する場合: プロジェクトルートに `requirements.txt` を作成し `pip install -r requirements.txt`)*

### 4. 設定 (Configuration)

プロジェクトルートに `.env` ファイルを作成し、ローカル開発用の環境変数を設定します。Lambdaデプロイ時は、Lambda関数の環境変数として設定します。

```env
# .env (ローカル開発用サンプル)

# AWS Configuration
AWS_REGION="ap-northeast-1"
BEDROCK_KNOWLEDGE_BASE_ID="YOUR_KNOWLEDGE_BASE_ID" # Bedrockコンソールで作成したナレッジベースID
BEDROCK_MODEL_ID_FAQ="anthropic.claude-3-sonnet-20240229-v1:0" # FAQ回答生成に使用するモデルID
# BEDROCK_MODEL_ID_EMBEDDING="amazon.titan-embed-text-v1" # これはナレッジベース作成時に指定

# External System Endpoints (サポートチケット起票支援API用 - アレンジ例)
# TICKET_SYSTEM_API_ENDPOINT="https://your-company.jira.com/rest/api/2/issue"
# TICKET_SYSTEM_API_KEY="YOUR_TICKET_SYSTEM_API_KEY_OR_TOKEN"

# Logging
LOG_LEVEL="INFO"
POWERTOOLS_SERVICE_NAME="SmartFAQBot" # aws-lambda-powertools用
```

### 5. データソースの準備 (FAQ機能)

1.  FAQの元となるドキュメント (PDF, TXT, DOCX, HTML, MDなど) を準備します。
2.  これらのドキュメントをAWS S3バケット (例: `s3://your-faq-documents-bucket/docs/`) にアップロードします。
3.  Amazon Bedrockコンソールでナレッジベースを作成します。
    *   データソースとして上記S3バケットを指定します。
    *   埋め込みモデルには `amazon.titan-embed-text-v1` を選択します。
    *   ベクトルストアには `Amazon OpenSearch Serverless` を選択し、設定します。
4.  作成したナレッジベースのIDを `.env` ファイルの `BEDROCK_KNOWLEDGE_BASE_ID` (ローカル用) およびLambdaの環境変数に設定します。

### 6. APIサーバーのローカル起動 (開発・テスト用)

```bash
poetry run uvicorn app.main:app --reload --port 8000
```
APIは `http://localhost:8000` で利用可能になります。Swagger UIは `http://localhost:8000/docs`、ReDocは `http://localhost:8000/redoc` でアクセスできます。
ローカルではCognito認証はスキップされるか、モック認証を実装する必要があります。

### 7. デプロイ (AWS Lambda + API Gateway + Cognito)

AWS SAM CLIまたはServerless Frameworkの使用を推奨します。

**AWS SAM を使用する場合 (例):**

1.  `template.yaml` (SAMテンプレート) をプロジェクトに合わせて記述します。Cognitoユーザープール、API Gateway、Lambda関数、IAMロールなどを定義します。
2.  ビルド: `sam build --use-container`
3.  デプロイ: `sam deploy --guided`

デプロイ後、API GatewayのエンドポイントURLが発行されます。Cognitoユーザープールでユーザーを作成し、払い出されたIDトークンをAuthorizationヘッダー (Bearerトークン) に含めてAPIを呼び出します。

## APIエンドポイント詳細

### 認証

API GatewayでCognitoオーソライザーを設定します。クライアントはCognitoから取得したIDトークンを `Authorization` ヘッダーにBearerトークンとして付与する必要があります。

```
Authorization: Bearer <YourCognitoIdToken>
```

### 7.1 FAQ問い合わせ

*   **エンドポイント:** `POST /faq/ask`
*   **説明:** 自然言語で質問を送信し、ナレッジベースから回答を取得します。
*   **リクエストボディ (JSON):**
    ```json
    {
      "question": "今年の夏季休暇はいつからいつまでですか？",
      "session_id": "optional_session_identifier_for_context" // オプション: 会話履歴管理用
    }
    ```
*   **レスポンス (JSON):**
    *   **成功時 (200 OK):**
        ```json
        {
          "question": "今年の夏季休暇はいつからいつまでですか？",
          "answer": "今年の夏季休暇は、8月13日（火）から8月16日（金）までの4日間です。詳細は社内ポータルの「就業規則第XX条」をご確認ください。",
          "source_chunks": [
            {
              "document_uri": "s3://your-faq-documents-bucket/HR/syugyou_kisoku_2024.pdf",
              "score": 0.89,
              "text": "夏季休暇: 毎年8月13日から16日までの4日間とする。"
            }
          ],
          "request_id": "lambda-request-id" // LambdaのリクエストID
        }
        ```
    *   **エラー時 (400 Bad Request, 401 Unauthorized, 403 Forbidden, 500 Internal Server Errorなど):**
        ```json
        {
          "detail": "エラーメッセージ" // FastAPIのデフォルトエラー形式
        }
        ```

### 7.2 サポートチケット起票支援 (※アレンジ例)

*   **エンドポイント:** `POST /support/create-ticket`
*   **説明:** FAQで解決しない問題について、ユーザーが記述した内容を基にサポートチケットの起票を支援します。
*   **リクエストボディ (JSON):**
    ```json
    {
      "problem_description": "PCの動作が非常に遅く、特定のアプリケーションが頻繁にフリーズします。再起動しても改善しませんでした。",
      "category": "IT_SUPPORT_HARDWARE", // オプション: 事前に定義されたカテゴリ
      "priority": "HIGH" // オプション: 緊急度
    }
    ```
*   **レスポンス (JSON):**
    *   **成功時 (201 Created - チケットシステムへ直接起票した場合):**
        ```json
        {
          "status": "ticket_created",
          "message": "サポートチケットが正常に起票されました。",
          "ticket_id": "JIRA-12345", // チケットシステム側のID
          "details": {
            "summary": "PC動作不良およびアプリケーションフリーズ", // LLMが生成または構造化した要約
            "description_provided": "PCの動作が非常に遅く、特定のアプリケーションが頻繁にフリーズします。再起動しても改善しませんでした。"
          },
          "request_id": "lambda-request-id"
        }
        ```
    *   **成功時 (200 OK - 起票用データを返却する場合):**
        ```json
        {
          "status": "data_prepared_for_ticket",
          "message": "サポートチケット起票用のデータ準備が完了しました。以下の内容で手動起票してください。",
          "ticket_data": { // チケットシステムに合わせた構造
            "project_key": "SUPPORT",
            "issue_type": "Bug",
            "summary": "PC動作不良およびアプリケーションフリーズ",
            "description": "ユーザー報告:\nPCの動作が非常に遅く、特定のアプリケーションが頻繁にフリーズします。再起動しても改善しませんでした。\n\n(システム追記: 関連FAQ検索結果なし)",
            "priority_name": "High"
          },
          "request_id": "lambda-request-id"
        }
        ```
    *   **エラー時:**
        ```json
        {
          "detail": "チケット起票支援に失敗しました。理由: [具体的なエラー理由]"
        }
        ```

## コード構成 (アレンジ例: Lambda + FastAPI)

```
.
├── app/                     # FastAPIアプリケーションのメインディレクトリ
│   ├── __init__.py
│   ├── main.py              # FastAPIアプリケーションインスタンス、Mangumハンドラ、ルーター読み込み
│   ├── routers/             # APIルーター (エンドポイント定義)
│   │   ├── __init__.py
│   │   └── faq.py
│   │   └── (support.py)     # (サポートチケット起票支援API用 - アレンジ例)
│   ├── services/            # ビジネスロジック、外部サービス連携
│   │   ├── __init__.py
│   │   └── bedrock_service.py
│   │   └── (ticket_service.py) # (サポートチケット起票支援API用 - アレンジ例)
│   ├── schemas/             # Pydanticモデル (リクエスト/レスポンススキーマ)
│   │   ├── __init__.py
│   │   └── faq.py
│   │   └── (support.py)     # (サポートチケット起票支援API用 - アレンジ例)
│   ├── core/                # 設定、共通ユーティリティ
│   │   ├── __init__.py
│   │   └── config.py        # 環境変数読み込み、設定クラス
│   └── utils/               # 汎用ユーティリティ関数
│       ├── __init__.py
│       └── logger.py        # LoguruまたはPowertools Loggerの設定
├── tests/                   # ユニットテスト、統合テスト
│   ├── __init__.py
│   ├── conftest.py
│   └── routers/
│   └── services/
├── .env.example             # 環境変数ファイルのサンプル (ローカル開発用)
├── .gitignore
├── poetry.lock
├── pyproject.toml           # Poetryプロジェクト定義
├── template.yaml            # (推奨: AWS SAMテンプレート)
├── README.md                # このファイル
└── (scripts/)               # (デプロイ用スクリプトなど)
```

## コストに関する注意 (Cost Considerations)

*   **Amazon Bedrock:** ナレッジベースのストレージ、モデルの推論 (FAQ回答生成、埋め込み生成) に対して料金が発生します。
*   **AWS Lambda:** リクエスト数、実行時間、メモリ割り当てに応じて料金が発生します。
*   **Amazon API Gateway:** リクエスト数、データ転送量に応じて料金が発生します。
*   **Amazon S3:** ドキュメントストレージ、ログ出力先として料金が発生します。
*   **Amazon OpenSearch Serverless:** インデックス作成ユニット、検索およびクエリユニット、データストレージに応じて料金が発生します。
*   **Amazon Cognito:** 月間アクティブユーザー数 (MAU) に応じて料金が発生する場合があります (無料利用枠あり)。
*   **CloudWatch:** ログ保存量、メトリクス、アラームに応じて料金が発生します。

コスト最適化のため、適切なモデルの選択、Lambdaのメモリ割り当ての最適化、不要なログ出力の抑制などを検討してください。AWS Cost Explorerで定期的にコストを確認することを推奨します。

## 今後の展望 (Future Work / ToDo)

*   **対話型FAQの強化:**
    *   Bedrock Agentsを活用したマルチターン会話への対応。
    *   ユーザーフィードバック機能 (回答の評価とナレッジベース改善サイクル)。
*   **業務自動化機能の拡充 (アレンジ例):**
    *   チケットシステムとのより高度な連携 (ステータス確認、更新など)。
    *   FAQ回答から直接関連ドキュメントへのリンクを提示するだけでなく、特定のアクション（例: ソフトウェア申請ページの表示）を実行する機能。
*   **運用・監視の強化:**
    *   `aws-lambda-powertools` を活用した構造化ログ、カスタムメトリクス、分散トレーシングの本格導入。
*   **テストカバレッジの向上。**

## コントリビューション (Contributing)

このプロジェクトへのコントリビューションを歓迎します！バグ報告、機能提案、プルリクエストはGitHubのIssuesやPull Requestsからお願いします。

## ライセンス (License)

このプロジェクトは [MIT License](LICENSE) の下で公開されています。
```

