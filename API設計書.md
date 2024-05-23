# HANABINOVATION API設計書
APIの構築にはAWS LambdaとAPI Gatewayを使用する。
APIにアクセスするURLは`https://${ドメイン名}/api/${API名}`。

## API一覧
API機能No. | 種別 | API名 | 機能概要
-|-|-|-
FIREWORKS-000|API|[fireworks](#fireworks)|花火データの送受信
PROFILES-000|API|[profiles](#profiles)|ユーザーデータの送受信

### fireworks
API機能No. | FIREWORKS-000
-|-
API名 | fireworks
概要 | 花火データの送受信
METHOD | GET or POST

#### fireworks GET
クエリ名 | 指定する型 | 指定する値 | クエリ概要
-|-|-|-
createdAfter | ISO 8601 | 特定の日時 | 特定の日時移行のデータに絞ってデータを取得する


##### 全データ取得
- 概要: 全ユーザーの花火データを取得する。
- アクセスURL: `api/fireworks`
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
- アクセスURL: `api/fireworks/${userId}`
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
- アクセスURL: `api/fireworks/${userId}/${boothId}`
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
- アクセスURL: `api/fireworks`
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
概要 | ユーザーデータの送受信
METHOD | GET or POST

#### profiles GET
##### 全データ取得
- 概要: 全ユーザーのデータを取得する。
- アクセスURL: `api/profiles`
- 取得データ
    ```ts
    {
        [userId: string]: userName: string; // ユーザーID: ユーザー名
    };
    ```

#### profiles POST
##### ユーザーデータ登録
- 概要: ユーザーデータを登録する。
- アクセスURL: `api/profiles`
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
