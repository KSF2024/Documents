# AWS手順書
## ドメインの取得
1. Route53でドメイン`hanabinovation.org`を取得する

## Reactフロントエンドの構築 (AWS Amplify)
1. AWS Amplifyコンソールに移動し、「新しいアプリをホスト」をクリックする
1. GitHubリポジトリを接続し、リポジトリとブランチを選択します。
1. 自動的にビルドとデプロイが行われ、公開URLが提供されます。

### カスタムドメインの設定
- カスタムドメインを追加する場合は、Amplifyの設定からRoute 53で取得したドメインを設定します。
1. 「すべてのアプリ -> リポジトリ名」から、リポジトリのホームメニューを開く
2. 画面左部メニューから、「ホスティング -> カスタムドメイン」をクリック
3. デプロイ済みのアプリのリダイレクト先に、Route53で取得したドメインを設定する

## Serverless Framework を Cloud9 にインストールする
1. Cloud9を起動する
1. ターミナルに以下のコマンドを入力して Serverless Framework(SLS)をインストールします。
    - sls ver4は認証が必要なので使いません。
    ```
    npm install -g serverless@3
    ```
2. Serverless Frameworkのプロジェクトを作成します。
    ```
    serverless create --template aws-nodejs --path hanabinovation
    cd hanabinovation
    ```
3. 後述する`serverless.yaml`と`handler.js`のソースコードの記述が完了したら、デプロイします
    ```
    serverless deploy
    ```

- 後程必要になるので`aws-sdk`もインストールします
    ```
    npm install @aws-sdk/client-dynamodb
    ```

### serverless.ymlの設定
```yaml
service: hanabinovation

provider:
  name: aws
  runtime: nodejs20.x
  region: ap-northeast-1

  environment:
    ACCOUNT_ID: { "Ref": "AWS::AccountId" }
    REGION: { "Ref": "AWS::Region" }

  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:PutItem
        - dynamodb:GetItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
        - dynamodb:Query
        - dynamodb:Scan
      Resource: 
        Fn::Sub: arn:aws:dynamodb:${self:provider.region}:${AWS::AccountId}:table/MyTable

functions:
  api:
    handler: handler.api
    events:
      - http:
          path: api
          method: get

  websocketHandler:
    handler: handler.websocketHandler
    events:
      - websocket:
          route: $default

resources:
  Resources:
    MyDynamoDBTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: MyTable
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 5
          WriteCapacityUnits: 5
```

### handler.jsの設定
サンプルコード
```js
const AWS = require('aws-sdk');
const dynamoDB = new AWS.DynamoDB.DocumentClient();

module.exports.api = async (event) => {
    const response = {
        statusCode: 200,
        body: JSON.stringify({ message: 'Hello from the API!' }),
    };
    return response;
};

module.exports.websocketHandler = async (event) => {
    const response = {
        statusCode: 200,
        body: 'Hello, WebSocket!',
    };
    return response;
};
```
