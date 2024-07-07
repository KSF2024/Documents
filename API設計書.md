# HANABINOVATION API設計書
APIの構築にはAWS LambdaとAPI Gatewayを使用する。
APIにアクセスするURLは`https://${ドメイン名}/api/v1/${API名}`。

## API一覧
API機能No. | 種別 | API名 | 機能概要
-|-|-|-
FIREWORKS-000|API|[fireworks](#fireworks)|花火データの送受信
PROFILES-000|API|[profiles](#profiles)|ユーザーデータの送信
LOTTERY-000|API|[lottery](#lottery)|当選者ユーザーの抽選
SEND_FIREWORKS-000|API|[sendFireworks](#sendFireworks)|花火データの一括送信

### fireworks
API機能No. | FIREWORKS-000
-|-
API名 | fireworks
概要 | 花火データの送受信
METHOD | GET, POST

#### fireworks GET
クエリ名 | 指定する型 | 指定する値 | クエリ概要
-|-|-|-
createdAfter | ISO 8601 | 特定の日時 | 特定の日時以降のデータに絞ってデータを取得する


##### 全データ取得
- 概要: 全ユーザーの花火データを取得する。
- アクセスURL: `api/v1/fireworks`
- 取得データ
    ```ts
    {
        [userId: string]: { // ユーザーID
            [boothId: string]: { // 各ブースのID
                createdAt: number; // データが登録された日時
                fireworkType: number; // 花火のセットアップの種類(0の場合はオリジナルデザインを使用)
                fireworkDesign: Blob; // ユーザーが作成した花火のオリジナルデザイン
                sparksType: number; // 火花のセットアップの種類
            };
        };
    };
    ```

##### ユーザーデータ取得
- 概要: 特定のユーザーの作成した、全ての花火のデータを取得する。
- アクセスURL: `api/v1/fireworks/${userId}`
- 取得データ
    ```ts
    {
        [boothId: string]: { // 各ブースのID
            createdAt: number; // データが登録された日時
            fireworkType: number; // 花火のセットアップの種類(0の場合はオリジナルデザインを使用)
            fireworkDesign: Blob; // ユーザーが作成した花火のオリジナルデザイン
            sparksType: number; // 火花のセットアップの種類
        };
    };
    ```

##### ブースデータ取得
- 概要: 特定のユーザーが、特定のブースで作成した花火のデータを1つ取得する。
- アクセスURL: `api/v1/fireworks/${userId}/${boothId}`
- 取得データ
    ```ts
    {
        createdAt: number; // データが登録された日時
        fireworkType: number; // 花火のセットアップの種類(0の場合はオリジナルデザインを使用)
        fireworkDesign: Blob; // ユーザーが作成した花火のオリジナルデザイン
        sparksType: number; // 火花のセットアップの種類
    };
    ```

#### fireworks POST
##### 花火データ登録
- 概要: 特定のブースで作成した花火のデータを登録する。
- アクセスURL: `api/v1/fireworks`
- 送信データ
    ```ts
    {
        userId: string; // ユーザーID
        boothId: string; // 各ブースのID
        fireworksData: {
            fireworkType: number; // 花火のセットアップの種類(0の場合はオリジナルデザインを使用)
            fireworkDesign: Blob; // ユーザーが作成した花火のオリジナルデザイン
            sparksType: number; // 火花のセットアップの種類
        };
    };
    ```

### profiles
API機能No. | PROFILES-000
-|-
API名 | profiles
概要 | ユーザーデータの送信
METHOD | GET, POST

#### profiles GET
##### 受付番号取得
- 概要: 受付番号を取得する。
- アクセスURL: `api/v1/profiles/${userId}`
- 送信データ
    ```ts
    {
        receipt: string; // 受付番号
        userName: string; // ユーザー名
    };
    ```

#### profiles POST
##### ユーザーデータ登録
- 概要: ユーザーデータを登録する。
    - ユーザーデータが登録されたユーザーには、受付番号`receipt`が発行されデータ登録が行われる。
        その後、「当選者ユーザーの抽選」において、抽選可能なユーザーとして扱われる。
- アクセスURL: `api/v1/profiles`
- 送信データ
    ```ts
    {
        userId: string, // ユーザーID
        userName: string; // ユーザー名
        email: string; // メールアドレス
        telephone: string; // 電話番号
        age: number; // 年齢
        schoolName: string; // 学校名
        schoolGrade: number; // 学年
    };
    ```

### lottery
API機能No. | LOTTERY-000
-|-
API名 | lottery
概要 | 当選者ユーザーの抽選
METHOD | GET, POST

#### lottery GET
##### ユーザー抽選
- 概要: 非当選者の中から1人、ユーザーのデータを取得する。
- アクセスURL: `api/v1/lottery`
- 取得データ
    ```ts
    {
        [receipt: string]: userName: string; // 受付番号: ユーザー名
    };
    ```

#### lottery POST
##### 当選ユーザー登録
- 概要: 指定したユーザーを当選者扱いにする。
    - 受信した受付番号`receipt`が合致するユーザーの`isWinner`を`true`にする。
- アクセスURL: `api/v1/lottery`
- 送信データ
    ```ts
    {
        receipt: string; // 受付番号
    };
    ```

### sendFireworks
API機能No. | SEND_FIREWORKS-000
-|-
API名 | sendFireworks
概要 | 花火データの一括送信
METHOD | POST

#### sendFireworks POST
##### 花火データの一括送信
- 概要: 指定したユーザーの登録済みの花火データを、websocketで一斉送信する
    - ユーザーIDでユーザーを指定し、POSTする部分のみこのAPIで行う
    - ユーザーIDを元にそのユーザーが登録した花火データを取得し、websocketで一斉送信する部分は「`receive-firework`」を参照
- アクセスURL: `api/v1/sendFireworks`
- 取得データ
    ```ts
    {
        userId: string; // ユーザーID
    };
    ```


### websocket
websocketの送信設計を以下に示す。

#### show-firework
- 概要: 登録があった花火のデータをwebsocketで送信する
    - `api/v1/fireworks`に花火データがPOSTされた際、そのデータをwebsocketで送信する。
- 送信データ
    ```ts
    {
        action: "show-firework";
        data: {
            boothId: string; // 各ブースのID
            fireworksData: {
                fireworkType: number; // 花火のセットアップの種類(0の場合はオリジナルデザインを使用)
                fireworkDesign: Blob; // ユーザーが作成した花火のオリジナルデザイン
                sparksType: number; // 火花のセットアップの種類
            };
        };
    };
    ```

#### send-fireworks
- 概要: 特定のユーザーが登録した全ての花火をwebsocketで送信する
    - `api/v1/sendFireworks`にユーザーIDがPOSTされた際、そのユーザーが登録した全ての花火データをwebsocketで送信する。
- 送信データ
    ```ts
    {
        action: "send-fireworks";
        data: {
            fireworksData: {
                [boothId: string] : { // 各ブースのID
                    fireworkType: number; // 花火のセットアップの種類(0の場合はオリジナルデザインを使用)
                    fireworkDesign: Blob; // ユーザーが作成した花火のオリジナルデザイン
                    sparksType: number; // 火花のセットアップの種類
                };
            };
        };
    };
    ```

#### draw-lottery
- 概要: 特定のユーザーが登録した全ての花火と応募受付情報をwebsocketで送信する
    - `api/v1/lottery`に当選確定者のユーザーIDがPOSTされた際、そのユーザーが登録した全ての花火データと応募受付情報をwebsocketで送信する。
- 送信データ
    ```ts
    {
        action: "draw-lottery";
        data: {
            userName: string; // ユーザー名
            receipt: string; // 受付番号
            fireworksData: {
                [boothId: string] : { // 各ブースのID
                    fireworkType: number; // 花火のセットアップの種類(0の場合はオリジナルデザインを使用)
                    fireworkDesign: Blob; // ユーザーが作成した花火のオリジナルデザイン
                    sparksType: number; // 火花のセットアップの種類
                };
            };
        };
    };
    ```
