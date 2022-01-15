---
title: "Twitter OAuth2.0の設定や動作まとめ"
emoji: "🐤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["oauth", "twitter"]
published: true
---

# はじめに
2021年12月15日にTwitterのOAuth2.0の正式提供開始のアナウンスがありました。
https://twitter.com/TwitterDevJP/status/1470916207130079239
本記事では上記の機能を利用したOAuth2.0 Clientの設定方法や挙動を記録して残します。
なお、Twitterは以下のようなOAuth 2.0 Authorization Code Flow with PKCEと呼ばれるフローを利用しています。
![](/images/8b1cfe654a1cee/pkce_flow.png)

OAuth2.0自体の説明は必要最低限としていますので、詳細については別途以下のOAuth2.0の仕様や記事などを参照いただけますと幸いです。
http://openid-foundation-japan.github.io/rfc6749.ja.html
https://datatracker.ietf.org/doc/html/rfc7636

本記事では説明の都合上、本来十分にランダムな値でなければならないstateやcode_verifier(およびcode_verifierから導出されるcode_challenge)について、固定の値を利用しています。
実際は推測不可能なランダムな文字列を利用するようにしてください。

また、Node.jsによる実装も用意しているので、あわせて参照いただけますと幸いです。
**Authorization Code Grant with PKCE:**
https://github.com/kg0r0/twitter-oauth2-client
**Client Credentials Grant:**
https://github.com/kg0r0/twitter-client-credentials


# Developer Portal 
Developer Portalにアクセスすると、``[User authentication settings]``という項目が追加されていることが確認できます。
(``[User authentication settings]``という設定ですが、2021/12/15時点ではOpenID Connectには対応しておらず、認証結果を安全に受け取ることができるわけではありません。あくまでTwitter APIにアクセスする場合に利用します。)
![](/images/8b1cfe654a1cee/developer_portal.png)
OAuth2.0を有効にした後に、``[OAUTH2.0 SETTINGS] > [Type of App]``から登録するアプリケーションのタイプを選択します。
![](/images/8b1cfe654a1cee/type_of_app.png)
``[Type of App]``の選択時、アプリケーションの種類の横にConfidential Client、Public Clientという記載があります。
これはOAuth2.0におけるClient Typeを表しています。OAuth2.0におけるConfidentil ClientとPublic Clientの定義はそれぞれ以下のとおりになっています。
- **Confidential Client**
クレデンシャルの機密性を維持することができるクライアント
- **Public Client**
クライアントクレデンシャルの機密性を維持することができず, かつ他の手段を使用したセキュアなクライアント認証もできないクライアント

(引用元: http://openid-foundation-japan.github.io/rfc6749.ja.html#client-types)

``[Keys and tokens] > [OAuth2.0 Client ID and Client Secret]``からClient IDとClient Secretが確認できます。
これらの値は後ほど利用するのでメモしておくようにします。なお、Client Secretは漏洩すると第三者が今回登録したClientになりすませるようになってしまうため、厳重に保管しておきます。
![](/images/8b1cfe654a1cee/client_id_secret.png)
``[User authentication settings] > [GENERAL AUTHENTICATION SETTINGS]``からリダイレクト後の戻り先URL(redirect_uri)を登録しておきます。
redirect_uriはOAuth1.0/2.0共通で以下のように設定することができます。
![](/images/8b1cfe654a1cee/redirect_uri.png)

# Confidential Client (WebApp / Authomated App or bot)

Confidential Client ("WebApp"または"Authomated App or bot"を選択した場合) の動作について解説します。

## Authorization Request / Response
Authorization Requestは以下のようになります。
```bash
https://twitter.com/i/oauth2/authorize?response_type=code&client_id=<Client ID>&redirect_uri=https://127.0.0.1:3000/cb&scope=tweet.read%20users.read%20offline.access&state=abc&code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&code_challenge_method=s256
```
Authorization Requestの各パラメータについて、OAuth2.0に準拠したパラメーターのみなので詳細は省きますが、簡単に補足すると以下の通りです。
- **response_type**
どのフローを利用するかを決定するパラメーター。Twitter OAuth2.0ではcode固定。
- **client_id**
Developer Portalで確認したクライアント識別子。
- **redirect_uri**
Developer Portalで登録したURL。
- **scope**
要求するアクセス範囲を明示するパラメーター。
[OAuth 2.0 Authorization Code Flow with PKCEのScopes](https://developer.twitter.com/en/docs/authentication/oauth-2-0/authorization-code)に記載されているものから選択する。
- **state**
CSRF対策用のパラメーター。(今回はサンプルなので推測可能な値を設定していますが、実際に利用する際はランダムな値を指定してください。)
- **code_challenge**
code_verifierを後述のcode_challenge_methodで計算したパラメーター。
なお、code_verifierは推測不可能なランダムな文字列であり、[RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636#section-4.1)には43文字から128文字の範囲で指定する必要がある旨が記載されています。
(今回は[RFC 7636の「Appendix B.  Example for the S256 code_challenge_method」](https://datatracker.ietf.org/doc/html/rfc7636#appendix-B)に記載されている値をそのまま利用しています。)
- **code_challenge_method**
code_verifierからcode_challengeを導出する際に利用するアルゴリズム。"plain"または"s256"が指定可能。

Authorization RequestのURLにアクセスすると、指定したscopeパラメーターにもとづいて以下のような画面が表示されます。
Clientが要求している権限が問題ない場合は``[Authorize app]``を押下して同意します。
![](/images/8b1cfe654a1cee/consent.png)
``[Authorize app]``を押下すると、以下のような事前に指定したリダイレクトURLにリダイレクトされます。
```bash
https://127.0.0.1:3000/cb?state=abc&code=<Authorization Code>
```
URL中のcodeパラメーターがAuthorization CodeなのでAccess Tokenを取得するために後ほど利用します。
また、Clientはここでstateパラメーターを検証します。

## Access Token Request / Response

さきほどのAuthorization Codeを利用して以下のようなリクエストを送信することでAccess Tokenを取得します。
```bash
curl --location --request POST 'https://api.twitter.com/2/oauth2/token' \
                  --basic -u '<Client ID>:<Client Secret>' \
                  --header 'Content-Type: application/x-www-form-urlencoded' \
                  --data-urlencode 'code=<Authorization Code>' \
                  --data-urlencode 'grant_type=authorization_code' \
                  --data-urlencode 'client_id=<Client ID>' \
                  --data-urlencode 'redirect_uri=https://127.0.0.1:3000/cb' \
                  --data-urlencode 'code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk'
{
  "token_type": "bearer",
  "expires_in": 7200,
  "access_token": "<Access Token>",
  "scope": "users.read tweet.read offline.access",
  "refresh_token": "<Refresh Token>"
}
```
Confidential Clientなので、AuthorizationヘッダのClient Secretに間違った値を指定すると以下のようにエラーとなることが確認できます。
```bash
{
  "error": "unauthorized_client",
  "error_description": "Missing valid authorization header"
}
```
なお、Authorizationヘッダにて既にClient IDを指定していますが、ボディにもClient IDを含める必要があり、含めない場合は以下の通りエラーとなります。
```bash
{
  "error": "invalid_request",
  "error_description": "Missing required parameter [client_id]."
}
```

## (余談) Client Credential Grantについて

Twitterは以前よりOAuth2.0のClient Credential Grantはサポートしています。
Confidential Clientであれば、用途によってClient Credential Grantの利用も選択肢に入るかと思います。
https://developer.twitter.com/en/docs/authentication/oauth-2-0/application-only
Client Credential Grantでは、別途発行されたConsumer KeysのAPI KeyとAPI Secretを利用します。
```bash
curl --location --request POST 'https://api.twitter.com/oauth2/token' \
                  --basic -u '<API Key>:<API Secret>' \
                  --header 'Content-Type: application/x-www-form-urlencoded;charset=UTF-8' \
                  --data-urlencode 'grant_type=client_credentials'
{
  "token_type": "bearer",
  "access_token": "<Access Token>"
}
```
OAuth2.0用に発行されたClient IDおよびClient Secretを用いてClient Credenial Grantは利用できず、以下のようなエラーになります。
これは、``[Type of App]``を"Web App"から"Automated App or bot"に変えた場合においても、特に変化は見受けらません。
```bash
curl --location --request POST 'https://api.twitter.com/2/oauth2/token' \
                  --basic -u '<Client ID>:<Client Secret>' \
                  --header 'Content-Type: application/x-www-form-urlencoded;charset=UTF-8' \
                  --data-urlencode 'grant_type=client_credentials' \
                  --data-urlencode 'client_id=<Client ID>'
{
  "error": "invalid_request",
  "error_description": "Value passed for the grant type was invalid. Grant type should be one of [authorization_code, refresh_token]."
}
```

なお、Authorization Code GrantとClient Credential GrantでToken Endpointが微妙に違う点には注意してください。
- Authorization Code Grant
https://api.twitter.com/2/oauth2/token
- Client Credential Grant
https://api.twitter.com/oauth2/token

# Public Client (Native App / Single page App)

Public Client ("Native App"または"Single page App"を選択した場合) の動作について解説します。

## Authorization Request / Response

Confidential Clientの際のAuthorization Request / Responseと異なる箇所はありません。
なお、Public Clientにおいてもscopeパラメーターにおいてoffline.accessを指定してRefresh Tokenを発行することができます。

## Access Token Request / Response
Access Token Request / Responseは以下のようになります。Authorizationヘッダによる認証が必要ない点がConfidential Clientの際と異なります。
```bash
 curl --location --request POST 'https://api.twitter.com/2/oauth2/token' \
                  --header 'Content-Type: application/x-www-form-urlencoded' \
                  --data-urlencode 'code=<Authorization Code> \
                  --data-urlencode 'grant_type=authorization_code' \
                  --data-urlencode 'client_id=<Client ID>' \
                  --data-urlencode 'redirect_uri=https://127.0.0.1:3000/cb' \
                  --data-urlencode 'code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk'
{
  "token_type": "bearer",
  "expires_in": 7200,
  "access_token": "<Access Token>",
  "scope": "users.read tweet.read offline.access",
  "refresh_token": "<Refresh Token>"
}
```
Refresh Tokenの利用についてはClient Typeに関係なく、Public Clientに対して発行されたRefresh Tokenも問題なく利用することができました。
```bash
curl --location --request POST 'https://api.twitter.com/2/oauth2/token' \
                 --basic -u '<Client ID>:<Client Secret>'\
                 --header 'Content-Type: application/x-www-form-urlencoded' \
                 --data-urlencode 'refresh_token=<Refresh Token>' \
                 --data-urlencode 'grant_type=refresh_token' \
                 --data-urlencode 'client_id=<Client ID>'
{
  "token_type": "bearer",
  "expires_in": 7200,
  "access_token": "<Access Token>",
  "scope": "users.read tweet.read offline.access",
  "refresh_token": "<Refresh Token>"
}
```
なお、Refresh Token利用時においても、ボディにもClient IDを含める必要があり、含めない場合は以下の通りエラーとなります。
```bash
{
  "error": "invalid_request",
  "error_description": "Missing required parameter [client_id]."
}
```
また、Public Client向けに発行されたRefresh Tokenを利用する場合においてもAuthorizationヘッダを付与する必要があり、付与しない場合は以下のようにエラーになります。
```bash
{
  "error": "unauthorized_client",
  "error_description": "Missing valid authorization header"
}
```
仕様上問題ない範囲内でPublic Clientに対してRefresh Tokenを発行する場合においても、漏洩した際に悪用されるリスクを考慮して設定、実装する必要があります。
本記事ではPublic Clientに対するのRefresh Tokenの発行についての詳細は扱いませんが、気になる方はOAuth 2.0 for Browser-Based Apps - 8. Refresh Tokensあたりをご覧いただくのが良いかと思います。
https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps-08.txt#section-8
なお、TwitterのOAuth2.0実装において、Refresh Tokenのローテーションが実装されていることは確認できました。

# おわりに

本記事では正式提供されたTwitter OAuth2.0についての設定方用や動作を解説しました。
正式提供されてあまり時間が経っておらず、ドキュメントも決して豊富ではないため、記載内容が誤っている箇所もあるかもしれません。
もし不備などを見つけた際には、PRやコメント、DMで報告いただけますと幸いです。
また、今後もTwitter OAuth2.0はアップデートされていくことが予見されるため、本記事の内容も逐次更新していければと思います。

# 参考
- https://twittercommunity.com/t/announcing-oauth-2-0-general-availability/163555
- https://developer.twitter.com/en/docs/authentication/oauth-2-0/authorization-code
- http://openid-foundation-japan.github.io/rfc6749.ja.html