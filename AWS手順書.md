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
    
  httpApi:
    cors:
      allowedOrigins:
        - "https://hanabinovation.org"
      allowedHeaders:
        - Content-Type
        - X-Amz-Date
        - Authorization
        - X-Api-Key
        - X-Amz-Security-Token
        - X-Amz-User-Agent
      allowedMethods:
        - OPTIONS
        - GET
        - POST
        - PUT
        - DELETE
      allowCredentials: true
    
  websocketsApiName: hanabinovationWebSocketApi
  websocketsApiRouteSelectionExpression: $request.body.action

  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:PutItem
        - dynamodb:GetItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
        - dynamodb:Query
        - dynamodb:Scan
        - execute-api:ManageConnections   # WebSocket APIの接続を管理する権限
        - execute-api:PostToConnection    # WebSocket APIに対してポスト操作を行う権限
      Resource: 
        - Fn::Sub: arn:aws:dynamodb:${self:provider.region}:${AWS::AccountId}:table/MyTable
        - Fn::Sub: arn:aws:dynamodb:${self:provider.region}:${AWS::AccountId}:table/WsMyTable

functions:
  getFireworks:
    handler: handler.getFireworks
    events:
      - http:
          path: api/v1/fireworks
          method: get
  getFireworksByUserId:
    handler: handler.getFireworksByUserId
    events:
      - http:
          path: api/v1/fireworks/{userId}
          method: get
  getFireworksByUserIdAndBoothId:
    handler: handler.getFireworksByUserIdAndBoothId
    events:
      - http:
          path: api/v1/fireworks/{userId}/{boothId}
          method: get
  postFireworks:
    handler: handler.postFireworks
    events:
      - http:
          path: api/v1/fireworks
          method: post
      - http:
          path: api/v1/fireworks
          method: options
  getProfiles:
      handler: handler.getProfiles
      events:
        - http:
            path: api/v1/profiles/{userId}
            method: get
  postProfiles:
    handler: handler.postProfiles
    events:
      - http:
          path: api/v1/profiles
          method: post
      - http:
          path: api/v1/profiles
          method: options
  getLottery:
    handler: handler.getLottery
    events:
      - http:
          path: api/v1/lottery
          method: get
  postLottery:
    handler: handler.postLottery
    events:
      - http:
          path: api/v1/lottery
          method: post
  sendFireworks:
    handler: handler.sendFireworks
    events:
      - http:
          path: api/v1/sendFireworks
          method: post
      - http:
          path: api/v1/sendFireworks
          method: options

  connect:
    handler: handler.connectHandler
    events:
      - websocket:
          route: $connect
  disconnect:
    handler: handler.disconnectHandler
    events:
      - websocket:
          route: $disconnect

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
    WsMyDynamoDBTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: WsMyTable
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
const { DynamoDBClient, ScanCommand, QueryCommand, PutItemCommand, GetItemCommand, UpdateItemCommand, DeleteItemCommand } = require("@aws-sdk/client-dynamodb");
const { ApiGatewayManagementApiClient, PostToConnectionCommand } = require("@aws-sdk/client-apigatewaymanagementapi");

const REGION = "ap-northeast-1";
const API_ID = "pcg2x1k5lj";
const STAGE = "dev";

// DynamoDBクライアントのインスタンスを作成
const client = new DynamoDBClient({ region: REGION });
const TABLE_NAME = "MyTable";

// websocket用のDynamoDBクライアントのインスタンスを作成
const wsClient = new DynamoDBClient({ region: REGION });
const WS_TABLE_NAME = "WsMyTable";
const WS_API_ENDPOINT = `https://${API_ID}.execute-api.${REGION}.amazonaws.com/${STAGE}`;

const HEADER = {
  "Access-Control-Allow-Origin": "https://hanabinovation.org",
  "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token,X-Amz-User-Agent",
  "Access-Control-Allow-Credentials": true
}

const PREFLIGHT_RESPONSE = {
  statusCode: 200,
  headers: HEADER,
  body: JSON.stringify({ message: "CORS preflight response" }),
};

// fireworks GET 全データ取得
exports.getFireworks = async (event) => {
  const { createdAfter } = event.queryStringParameters || {};
  const params = {
    TableName: TABLE_NAME
  };

  const result = {};
  const allItems = [];
  try {
    const data = await client.send(new ScanCommand(params));

    data.Items.forEach(userData => {
      const userId = userData.id.S;

      const fireworksData = userData.fireworksData?.M;
      if (!fireworksData) return;

      // 全データを取得する
      Object.keys(fireworksData).forEach(boothId => {
        const boothData = fireworksData[boothId]?.M;
        if (!boothData) return;

        const createdAt = boothData.createdAt?.N || 0;
        if (!createdAfter || createdAt > new Date(String(createdAfter)).getTime()) {
          const newData = {
            userId: userId,
            boothId: boothId,
            createdAt: Number(boothData.createdAt.N),
            fireworkType: Number(boothData.fireworkType.N),
            sparksType: Number(boothData.sparksType.N)
          }
          if (boothData.fireworkDesign?.S) {
            newData.fireworkDesign = boothData.fireworkDesign.S;
          }

          allItems.push(newData);
        }
      });
    });

    // createdAtでソートして最新の100個を取得
    const latestItems = allItems.sort((a, b) => b.createdAt - a.createdAt).slice(0, 100);

    // 結果を構造化
    latestItems.forEach(item => {
      if (!result[item.userId]) result[item.userId] = {};
      result[item.userId][item.boothId] = {
        createdAt: item.createdAt,
        fireworkType: item.fireworkType,
        sparksType: item.sparksType,
        ...(item.fireworkDesign && { fireworkDesign: item.fireworkDesign })
      };
    });

    return {
      statusCode: 200,
      headers: HEADER,
      body: JSON.stringify(result),
    };
  } catch (error) {
    console.error(error);
    return {
      statusCode: 500,
      headers: HEADER,
      body: JSON.stringify({ message: 'Internal Server Error'}),
    };
  }
};

// fireworks GET ユーザーデータ取得
exports.getFireworksByUserId = async (event) => {
  const { userId } = event.pathParameters;
  const params = {
    TableName: TABLE_NAME,
    Key: { id: { S: userId } },
    ProjectionExpression: "fireworksData"
  };

  try {
    const fireworksData = await getFireworksDataByUserId(userId);
    if(fireworksData){
      return {
        statusCode: 200,
        headers: HEADER,
        body: JSON.stringify(fireworksData),
      };      
    }else{
      return {
        statusCode: 404,
        headers: HEADER,
        body: JSON.stringify({ message: 'Data not found' }),
      };
    }
  }catch(error){
    console.error(error)
    return {
      statusCode: 500,
      headers: HEADER,
      body: JSON.stringify({ message: 'Internal Server Error' }),
    };
  }
};

// fireworks GET ブースデータ取得
exports.getFireworksByUserIdAndBoothId = async (event) => {
  const { userId, boothId } = event.pathParameters;
  const params = {
    TableName: TABLE_NAME,
    Key: { id: { S: userId } },
    ProjectionExpression: `fireworksData.${boothId}`
  };

  try {
    const data = await client.send(new GetItemCommand(params));
    const fireworkData = data.Item?.fireworksData?.M?.[boothId]?.M;
    if(fireworkData){
      const result = {
        createdAt: fireworkData.createdAt.N,
        fireworkType: fireworkData.fireworkType.N,
        sparksType: fireworkData.sparksType.N
      };
      if(fireworkData.fireworkDesign?.S){
        result.fireworkDesign = fireworkData.fireworkDesign.S
      }
      return {
        statusCode: 200,
        headers: HEADER,
        body: JSON.stringify(result),
      };
    }else{
      return {
        statusCode: 404,
        headers: HEADER,
        body: JSON.stringify({ message: 'Data not found' }),
      };
    }
  }catch(error){
    console.error(error)
    return {
      statusCode: 500,
      headers: HEADER,
      body: JSON.stringify({ message: 'Internal Server Error' }),
    };
  }
};

// fireworks POST 花火データ登録
exports.postFireworks = async (event) => {
  // プリフライトリクエストを処理する
  if (event.httpMethod === "OPTIONS") return PREFLIGHT_RESPONSE;
  
  const body = JSON.parse(event.body);

  // データベースに登録するための花火のデータを取得する
  const fireworksData = {
    createdAt: { N: Date.now().toString() },
    fireworkType: { N: body.fireworksData.fireworkType.toString() },
    sparksType: { N: body.fireworksData.sparksType.toString() }
  };

  if (body.fireworksData.fireworkDesign) {
    fireworksData.fireworkDesign = { S: body.fireworksData.fireworkDesign };
  }
  
  // ネストされたデータを作成するための親要素があるかどうかを確認する
  try {
    const scanParams = {
      TableName: TABLE_NAME,
      Key: { id: { S: body.userId } }
    };
    const data = await client.send(new GetItemCommand(scanParams));
    
    // 親要素がない場合は、親要素を作成する
    const exsistsFireworksData = data.Item?.fireworksData;
    if(!exsistsFireworksData){
      // ネストされたデータを作成するための更新式を定義する
      const preparationParams = {
        TableName: TABLE_NAME,
        Key: { id: { S: body.userId } },
        UpdateExpression: "SET #fireworksData = :emptyMap",
        ExpressionAttributeNames: { "#fireworksData": "fireworksData" },
        ExpressionAttributeValues: { ":emptyMap": { M: {} } }
      };
  
      await client.send(new UpdateItemCommand(preparationParams));
    }
  }catch(e){
    console.error("scanError: ", e);
  }

  // ネストされた更新式を定義する
  const params = {
    TableName: TABLE_NAME,
    Key: { id: { S: body.userId } },
    UpdateExpression: "SET #fireworksData.#boothId = :data",
    ExpressionAttributeNames: {
      "#fireworksData": "fireworksData",
      "#boothId": body.boothId,
    },
    ExpressionAttributeValues: { ":data": { M: fireworksData } }
  };

  try {
    // UpdateItemCommandの実行
    await client.send(new UpdateItemCommand(params));
    
    // データベースに保存した花火データを、websocketでメッセージとして送信する
    const message = {
      action: "show-firework",
      data: {
        boothId: body.boothId,
        fireworksData: body.fireworksData
      }
    };
    await broadcastMessage(message);

    return {
      statusCode: 200,
      headers: HEADER,
      body: JSON.stringify({ message: 'Fireworks data updated successfully' }),
    };
  } catch (error) {
    console.error(error);
    return {
      statusCode: 500,
      headers: HEADER,
      body: JSON.stringify({ message: 'Internal Server Error' }),
    };
  }
};

// profiles GET ユーザーデータ取得
exports.getProfiles = async (event) => {
  const { userId } = event.pathParameters;
  const params = {
    TableName: TABLE_NAME,
    Key: { id: { S: userId } },
    ProjectionExpression: "profile"
  };

  try {
    const data = await client.send(new GetItemCommand(params));
    const profileData = data.Item?.profile?.M;
    if(profileData){
      const result = {
        receipt: profileData.receipt.S,
        userName: profileData.userName.S
      };
      return {
        statusCode: 200,
        headers: HEADER,
        body: JSON.stringify(result),
      };
    }else{
      return {
        statusCode: 404,
        headers: HEADER,
        body: JSON.stringify({ message: 'Data not found' }),
      };
    }
  }catch(error){
    console.error(error);
    return {
      statusCode: 500,
      headers: HEADER,
      body: JSON.stringify({ message: 'Internal Server Error' }),
    };
  }
};

// profiles POST ユーザーデータ登録
// profileの中身を上書きする(一部パラメータのみの書き換えには非対応のため注意)
exports.postProfiles = async (event) => {
  // プリフライトリクエストを処理する
  if (event.httpMethod === "OPTIONS") return PREFLIGHT_RESPONSE;
  
  const body = JSON.parse(event.body);
  
  const newReceipt = generateReceipt();
  const profile = {
    userName: { S: body.userName },
    email: { S: body.email },
    receipt: { S: newReceipt },
    isWinner: { BOOL: false }
  };
  
  if (body.telephone) profile.telephone = { S: body.telephone };
  if (body.age) profile.age = { N: body.age.toString() };
  if (body.schoolName) profile.schoolName = { S: body.schoolName };
  if (body.schoolGrade) profile.schoolGrade = { N: body.schoolGrade.toString() };
  
  const updateExpression = "SET #profile = :profile";
  const expressionAttributeNames = { "#profile": "profile" };
  const expressionAttributeValues = { ":profile": { M: profile } };

  const params = {
    TableName: TABLE_NAME,
    Key: { id: { S: body.userId } },
    UpdateExpression: updateExpression,
    ExpressionAttributeNames: expressionAttributeNames,
    ExpressionAttributeValues: expressionAttributeValues
  };

  try {
    await client.send(new UpdateItemCommand(params));
    return {
      statusCode: 200,
      headers: HEADER,
      body: JSON.stringify({ message: 'Profile added successfully', receipt: newReceipt }),
    };
  } catch (error) {
    return {
      statusCode: 500,
      headers: HEADER,
      body: JSON.stringify({ message: 'Internal Server Error' }),
    };
  }
};

// lottery GET ユーザー抽選
exports.getLottery = async (event) => {
  const params = {
    TableName: TABLE_NAME,
    FilterExpression: "profile.isWinner = :false",
    ExpressionAttributeValues: {
      ":false": { BOOL: false }
    }
  };

  try {
    const data = await client.send(new ScanCommand(params));
    if (data.Items.length > 0) {
      const winner = data.Items[Math.floor(Math.random() * data.Items.length)];
      const result = {
        [winner.profile.M.receipt.S]: winner.profile.M.userName.S
      };
      return {
        statusCode: 200,
        headers: HEADER,
        body: JSON.stringify(result),
      };
    } else {
      return {
        statusCode: 404,
        headers: HEADER,
        body: JSON.stringify({ message: 'No eligible users found' }),
      };
    }
  } catch (error) {
    return {
      statusCode: 500,
      headers: HEADER,
      body: JSON.stringify({ message: 'Internal Server Error' }),
    };
  }
};

// lottery POST 当選ユーザー登録
exports.postLottery = async (event) => {
  // プリフライトリクエストを処理する
  if (event.httpMethod === "OPTIONS") return PREFLIGHT_RESPONSE;
  
  const body = JSON.parse(event.body);
  const { receipt } = body;

  const scanParams = {
    TableName: TABLE_NAME,
    FilterExpression: "profile.receipt = :receipt",
    ExpressionAttributeValues: {
      ":receipt": { S: receipt }
    }
  };

  try {
    const scanResult = await client.send(new ScanCommand(scanParams));

    if (scanResult.Items.length === 0) {
      return {
        statusCode: 404,
        headers: HEADER,
        body: JSON.stringify({ message: "No user found with the provided receipt." }),
      };
    }

    const user = scanResult.Items[0];
    const userId = user.id.S;

    const updateParams = {
      TableName: TABLE_NAME,
      Key: { id: { S: userId } },
      UpdateExpression: "SET profile.isWinner = :isWinner",
      ExpressionAttributeValues: {
        ":isWinner": { BOOL: true }
      }
    };

    // データベース上の当選状況を"当選済み"に設定する
    await client.send(new UpdateItemCommand(updateParams));
    
    // 当選が確定したユーザーの花火データをwebsocketで全体送信する
    const fireworksData = await getFireworksDataByUserId(userId);
    const message = {
      action: "draw-lottery",
      data: {
        userName: scanResult.Items[0].profile.M.userName.S,
        receipt,
        fireworksData
      }
    };
    await broadcastMessage(message);

    return {
      statusCode: 200,
      headers: HEADER,
      body: JSON.stringify({ message: "User status updated successfully" }),
    };
  } catch (error) {
    console.error(error);
    return {
      statusCode: 500,
      headers: HEADER,
      body: JSON.stringify({ message: "Internal Server Error" }),
    };
  }
};

// sendFireworks POST 花火データの一括送信
exports.sendFireworks = async (event) => {
    // プリフライトリクエストを処理する
  if (event.httpMethod === "OPTIONS") return PREFLIGHT_RESPONSE;
  
  const body = JSON.parse(event.body);
  const userId = body.userId;

  try {
    // 送信されたユーザーIDから、指定されたユーザーが作成した花火データを取得する
    const fireworksData = await getFireworksDataByUserId(userId);
    if(fireworksData){
      // 取得した花火データを、websocketでメッセージとして送信する
      const message = {
        action: "send-fireworks",
        data: {
          fireworksData
        }
      };
      await broadcastMessage(message);
      
      return {
        statusCode: 200,
        headers: HEADER,
        body: JSON.stringify({ message: 'Fireworks data sent successfully' }),
      };
    }else{
      return {
        statusCode: 404,
        headers: HEADER,
        body: JSON.stringify({ message: 'Data not found' }),
      };
    }
  }catch (error){
    console.error(error);
    return {
      statusCode: 500,
      headers: HEADER,
      body: JSON.stringify({ message: 'Internal Server Error' }),
    };
  }
};


// websocket処理
// websocket connect
exports.connectHandler = async (event) => {
  const connectionId = event.requestContext.connectionId;

  const params = {
    TableName: WS_TABLE_NAME,
    Item: {
      id: { S: connectionId }
    }
  };

  try {
    await client.send(new PutItemCommand(params));
    return {
      statusCode: 200,
      // headers: HEADER,
      body: 'Connected'
    };
  } catch (error) {
    console.error(error);
    return {
      statusCode: 500,
      // headers: HEADER,
      body: 'Failed to connect: ' + JSON.stringify(error)
    };
  }
};

// websocket disconnect
exports.disconnectHandler = async (event) => {
  const connectionId = event.requestContext.connectionId;

  const params = {
    TableName: WS_TABLE_NAME,
    Key: {
      id: { S: connectionId }
    }
  };

  try {
    await client.send(new DeleteItemCommand(params));
    return {
      statusCode: 200,
      // headers: HEADER,
      body: 'Disconnected'
    };
  } catch (error) {
    console.error(error);
    return {
      statusCode: 500,
      // headers: HEADER,
      body: 'Failed to disconnect: ' + JSON.stringify(error)
    };
  }
};


// 指定のユーザーにwebsocketのメッセージを送信する関数
async function sendWebSocketMessage(connectionId, message){
  const wsClient = new ApiGatewayManagementApiClient({ endpoint: WS_API_ENDPOINT });

  try {
    await wsClient.send(new PostToConnectionCommand({
      ConnectionId: connectionId,
      Data: JSON.stringify(message),
    }));
  } catch (error) {
    console.error("Failed to send message:", error);
  }
};

// 現在接続中のすべてのユーザーにwebsocketのメッセージを送信する関数
async function broadcastMessage(message){
  const params = {
    TableName: WS_TABLE_NAME
  };

  try {
    const data = await client.send(new ScanCommand(params));
    for (const item of data.Items) {
      const connectionId = item.id.S;
      console.log("send: ", connectionId);
      if(connectionId) await sendWebSocketMessage(connectionId, message);
    }
  } catch (error) {
    console.error("Failed to broadcast message:", error);
  }
};

// 指定のユーザーが作成した花火データを取得する関数
async function getFireworksDataByUserId(userId){
    const params = {
      TableName: TABLE_NAME,
      Key: { id: { S: userId } },
      ProjectionExpression: "fireworksData"
    };
  
    const data = await client.send(new GetItemCommand(params));
    const fireworkData = data.Item?.fireworksData?.M;
    if(fireworkData){
      const result = {};
      Object.keys(fireworkData).forEach(boothId => {
        const boothData = fireworkData[boothId].M;
        result[boothId] = {
          createdAt: Number(boothData.createdAt.N),
          fireworkType: Number(boothData.fireworkType.N),
          sparksType: Number(boothData.sparksType.N)
        };
        if(boothData.fireworkDesign?.S){
          result[boothId].fireworkDesign = boothData.fireworkDesign.S;
        }
      });
      
      return result;
    }else{
      return null;
    }
}

// 一意な抽選会応募受付番号を生成する関数
function generateReceipt() {
  // 現在のタイムスタンプを取得し、文字列に変換
  const timestamp = Date.now().toString(36);

  // ランダムな文字列を生成
  const randomString = Math.random().toString(36).substr(2, 12);

  // タイムスタンプとランダムな文字列を組み合わせ、10〜20文字の長さに調整
  const receipt = timestamp + "-" + randomString;

  // 必要に応じて切り取る
  return receipt.toUpperCase();
  // return receipt.toUpperCase().substring(0, 20);
}
```
