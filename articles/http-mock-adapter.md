---
title: '[dio × riverpod] http_mock_adapter で APIクライアントのテストを書く'
emoji: '🧪'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [flutter, dart, riverpod, test]
published: true
---

<!-- この記事は [株式会社 TORICO Advent Calendar 2022](https://qiita.com/advent-calendar/2022/torico) 19 日目の記事です 🎄 -->

## 🤔 **http_mock_adapter** とは？

@[card](https://pub.dev/packages/http_mock_adapter)

http_mock_adapter は、テストで使用することを目的とした [dio](https://pub.dev/packages/dio) を用いた HTTP 通信のモッキングができるパッケージです。

宣言的にモッキングできるので、扱いやすくておすすめです 🎁

## インストール

パッケージの [Readme](https://pub.dev/packages/http_mock_adapter) に記されている手順でパッケージをインストールします。

1. **http_mock_adapter** を pubspec.yaml の dev_dependencies に追加
2. `flutter pub get` を実行

#### 💡VSCode の Flutter 拡張機能を使った `pub add`

![](/images/clean-shot-2022-12-18-at-14-50-48.gif.gif)

1. コマンドパレットを開く（Shift + Command + P）
2. コマンドパレットに dart と入力
3. Dart: Add Dev Dependency を選択
4. 追加したいパッケージ名を入力もしくは、選択して return

## http_mock_adapter の使い方

パッケージの Readme に記載されているサンプルコードにコメントを加える形で、http_mock_adapter の使い方を解説します。

```dart
import 'package:dio/dio.dart';

// ①： パッケージをインポート
import 'package:http_mock_adapter/http_mock_adapter.dart';

void main() async {
  // ②： モック化したい HTTP通信のクライアントを担う Dio インスタンス。
  final dio = Dio(BaseOptions());

  // ③： ②をコンストラクタに渡して、DioAdapter をインスタンス化。
  final dioAdapter = DioAdapter(dio: dio);

  // ④： モック化するリクエスト先のパス。
  const path = 'https://example.com';

  // ⑤： ②の dio の get メソッドによる、HTTP通信のモッキング設定を行う。
  dioAdapter.onGet(
    path,
    (server) => server.reply(
      200,
      {'message': 'Success!'},
    ),
  );

  // ⑥： ⑤で、モックレスポンスの設定をしたパスに、②の dio の get でリクエストする。
  final response = await dio.get(path);

  // ⑥： ④で設定した通りのレスポンスが返ってきている 🙌
  print(response.data); // {message: Success!}
}
```

`DioAdapter` は http_mock_adapter パッケージに定義されているクラスで、

`DioAdapter` の `onGet` メソッドの第一引数で渡したパスに、`DioAdapter` のコンストラクタに渡した `Dio` インスタンスが `get` メソッドでリクエストした際に返ってくる、モックレスポンスのステータスコード、レスポンスボディ、そのリクエストに必要なリクエストボディ、クエリパラメータ等を指定することができます。

`onGet` 以外にも、`post`, `put`, `delete` に対応する、`onPost` , `onPut` ,`onDelete` 等も定義されています 🙌

## Riverpod で Dio を DI した API クライアントのユニットテスト

Dio を用いた HTTP 通信時のヘッダーに付与する情報、エラーハンドリング、 レスポンスボディで返される JSON のパース処理などを共通化するために、アプリケーション内で独自の API クライアントクラスを定義して使うことがあると思います。

以下は、[riverpod](https://pub.dev/packages/riverpod) パッケージの [Provider](https://docs-v2.riverpod.dev/docs/providers/provider) を使って、Dio インスタンスを DI した API クライアントクラスのユニットテストの例です 🧪

```dart
import 'dart:io';

// 依存パッケージ
import 'package:dio/dio.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:http_mock_adapter/http_mock_adapter.dart';

// 趣味で開発している、旅行計画の作成・管理アプリ。
import 'package:trip_app_nativeapp/core/exception/api_exception.dart';
import 'package:trip_app_nativeapp/core/http/api_client/api_client.dart';
import 'package:trip_app_nativeapp/core/http/api_client/api_destination.dart';
import 'package:trip_app_nativeapp/core/http/api_client/dio/dio.dart';

Future<void> main() async {
  group(
    // [ApiClient] アプリケーション内に定義している、独自のAPIクライアント
    'ApiClient test',
    () {
      // setUp()内で初期化して、group()内の全てのテストケースで使用するため
      // lateキーワードを付け、ここで宣言。
      late DioAdapter dioAdapter;
      late ProviderContainer container; // Provider が持つ状態を保持するコンテナ

      const dummyEndpoint = '/dummy-endpoint';

      // setUp は group内の全てのテスト実行前に呼び出される。
      setUp(
        () {
          // テスト用の Dio をインスタンス化
          final dio = Dio(
            // validateStatus のコールバックで true を返すことで
            // dio が受け取った HTTPレスポンスが、いかなるステータスコードであっても、
            // DioError が投げられず、Response が返されるようにしている。
            BaseOptions(validateStatus: (status) => true),
          );

          dioAdapter = DioAdapter(dio: dio);

          container = ProviderContainer(
            overrides: [
              // [dioProvider] Dio インスタンスを提供する Provider
              // ApiClient に Dio を DIするのに使う。
              // テスト用の dio インスタンスで、dioProvider が提供する値をオーバーライドする。
              dioProvider(ApiDestination.privateTripAppV1)
                  .overrideWithValue(dio),
            ],
          );
        },
      );

      test(
        'post 正常系',
        // 省略
      );

      // [ApiException] アプリケーション内に定義している、独自の例外クラス。
      test(
        'post 準正常系 ステータスコードが 401 の場合は ApiException が スローされるはず。',
        () async {
          dioAdapter.onPost(
            dummyEndpoint,
            (server) => server.reply(
              HttpStatus.unauthorized, // 401
              <String, dynamic>{
                'error_code': 'test',
                'description': 'test',
              },
            ),
          );
          await expectLater(
            () async {
              // [privateTripAppV1ClientProvider] ログイン認証情報を必要とする
              // エンドポイントにリクエストする ApiClient を提供する Provider
              await container.read(privateTripAppV1ClientProvider).post(
                    dummyEndpoint,
                  );
            },
            // post メソッドでリクエストして、ステータスコード401のレスポンスを受けた際に
            // ApiClient が投げる ApiException のフィールドの値と
            // dioAdapter.onPost で設定した、モックの値が一致するか否かを検証。
            throwsA(
              isA<ApiException>()
                  .having(
                    (e) => e.statusCode,
                    'statusCode',
                    HttpStatus.unauthorized,
                  )
                  .having(
                    (e) => e.errorCode,
                    'errorCode',
                    'test',
                  )
                  .having(
                    (e) => e.description,
                    'description',
                    'test',
                  ),
            ),
          );
        },
      );
    },
  );
}

```

以上です！最後まで読んでくださってありがとうございます ( ◜ ◡ ◝ ) 🫧

上記のサンプルコードの全体はこちらから見れます 👀

https://github.com/seigi0714/trip-app-nativeapp/blob/main/test/core/http/api_client/api_client_test.dart

## 🙏 References

https://pub.dev/packages/http_mock_adapter

https://docs-v2.riverpod.dev/ja/docs/cookbooks/testing

https://shan-shaji.medium.com/mock-dio-in-flutter-f7f97082135f

https://github.com/salvadordeveloper/flutter-crypto-app/blob/main/test/api_test.dart
