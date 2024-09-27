---
title: "Amazon CognitoとOpenID Connect Providerを連携してアカウントをリンクする"
emoji: "👥"
type: "tech"
topics:
  - "oidc"
  - "cognito"
published: true
published_at: "2024-01-23 16:33"
---

[Amazon CognitoとOpenID Connect Providerを連携する](https://zenn.dev/muramasa64/articles/33a1c18db96229)の続き。CognitoにはUser Poolのアカウントと外部のプロバイダーから連携してきたアカウントと同一のアカウントとしてみなせる機能があるので、それを使う方法について簡単にまとめる。

## 前提条件

- CognitoとOIDC Provider（OP）の連携済みであること
- 連携対象のOPは、前回の記事と同様に[PingOne for Enterprise](https://developer.pingidentity.com/en/cloud-services/pingone-for-enterprise.html)とする

## 手順

### CognitoのUser Poolにアカウントを登録する

まずは、User Poolにアカウント（ユーザー）を作成する。

```sh
aws cognito-idp admin-create-user --user-pool-id ${USER_POOL_ID} --username ${USERNAME} --user-attributes Name=email,Value=${EMAIL} Name=email_verified,Value=true --message-action SUPPRESS
```

それぞれのパラメータについての説明

- `--user-pool-id`
	- ユーザーを作成する対象となるUser Pool ID
- `--username`
	- 作成するユーザーの認証に使うIDを指定する。
	- 今回の場合はユーザーのEメールアドレスを使う。
- `--user-attributes`
	- 作成するユーザーの属性を個別に登録する
	- `email`
		- ここでは、usernameとは別に、email属性にEメールアドレスを登録する
	- `email_verified`
		- 今回は、OPからフェデレーションしてくるユーザーのメールアドレスはすでに検証済みという前提とするため、Eメールアドレスは検証済みであることを示すフラグを立てる
- `--message-action`
	- ユーザーを作成した後、対象となるユーザーに対して、Eメールで通知を送るかどうかを決める。
	- `SUPPRESS`を指定することで、通知しないようにする。

ユーザーの作成が成功すると、次のような結果が返る[^1][^2]。

[^1]: AWS CLIのパラメータで`--username`としてメールアドレスを指定しているが、返ってきた`User.Username`の値は、UUIDになっているのに、やや混乱する。この辺りは、Cognitoの機能としてサインインに利用できるユーザー名（識別子）を選ぶことができる機能の影響かと思われる。この辺りについては別途整理する。

[^2]: このユーザーに対してパスワードが未設定になっているので`User.UserStates`が、`FORCE_CHANGE_PASSWORD`となっているが、今回のケースではパスワードで認証する必要がなく、また、パスワードで認証できない状態なので無視して良い。


```json
{
    "User": {
        "Username": "6d314364-9ca2-46f9-b954-b6764690f900",
        "Attributes": [
            {
                "Name": "sub",
                "Value": "6d314364-9ca2-46f9-b954-b6764690f900"
            },
            {
                "Name": "email_verified",
                "Value": "true"
            },
            {
                "Name": "email",
                "Value": "user@example.com"
            }
        ],
        "UserCreateDate": "2024-01-23T14:25:46.267000+09:00",
        "UserLastModifiedDate": "2024-01-23T14:25:46.267000+09:00",
        "Enabled": true,
        "UserStatus": "FORCE_CHANGE_PASSWORD"
    }
}
```

### 作成したユーザーと外部ユーザーをリンクする

User Poolに作成したユーザーと、外部プロバイダーのアカウントを連携するには、`admin-link-provider-for-user`を実行する。ちなみに、このオペレーションはAWS Management Consoleで実施することができない。

また、対象となるユーザーはOPを経由したサインインをする前に、これを実行する必要がある。もしすでに外部アカウントがUser Poolに登録されてしまっているなら、そのアカウントを先に削除すること。

```sh
aws cognito-idp admin-link-provider-for-user \
  --user-pool-id ${USER_POOL_ID} \
  --source-user ProviderName=${PROVIDER_NAME},ProviderAttributeName=email,ProviderAttributeValue=user@example.com \
  --destination-user ProviderName=Cognito,ProviderAttributeValue=${USERNAME}
```

それぞれのパラメータについての説明
- `--user-pool-id`
	- 対象となるUser Pool ID
- `--source-user`
	- リンク元となる外部プロバイダー（OP）のユーザーの、リンクに必要となる属性名とその値を指定する。
	- `ProviderName`
		- リンク対象となるプロバイダーの名前[^3]
	- `ProviderAttributeName`
		- プロバイダーから取得できる属性の名前
		- ここでは`email`としている。実際には一意に特定できるを使うこと
	- `ProviderAttributeValue`
		- リンク対象となる値を指定する
		- ここではユーザーのメールアドレスの値（user@example.com）としている
- `--destination-user`
	- Cognitoのユーザーを特定する値を指定する
	- `ProviderName`
		- `Cognito`とする
	- `ProviderAttributeValue
		- Cognitoの`Username`を指定する。
		- ユーザーを作成したときに出力されたJSONの`User.Username`の値とする
		- ここでは、UUIDの`6d314364-9ca2-46f9-b954-b6764690f900`となる


[^3]: AWS CLIで取得するには、下記のようにする。`aws cognito-idp list-identity-providers --user-pool-id $USER_POOL_ID --query 'Providers[].ProviderName'`

このコマンドの実行に成功したら特に出力はないので、確認のため対象となるユーザーの情報を取得する。

```sh
aws cognito-idp admin-get-user --user-pool-id ${USER_POOL_ID} --username ${USERNAME} --query 'UserAttributes[?Name==`identities`]'
```

下記のような出力が得られる。

```json
[
    {
        "Name": "identities",
        "Value": "[{\"userId\":\"user@example.com\",\"providerName\":\"PingOneOIDC\",\"providerType\":\"OIDC\",\"issuer\":null,\"primary\":false,\"dateCreated\":1705991781374}]"
    }
]
```

この`identities`属性の値は、通常の方法では変更不可で、リンクの設定が格納されている。このJSONのエスケープを解除するとこうなる。

```json
[
    {
        "userId": "user@example.com",
        "providerName": "PingOneOIDC",
        "providerType": "OIDC",
        "issuer": null,
        "primary": false,
        "dateCreated":1705991781374
    }
]
```

これにより、対象となるプロバイダーからemailが`user@example.com`のユーザーがサインインしてきたら、リンク設定したユーザーとしてサインインされるようになる。

