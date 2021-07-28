---
header: "Microsoft Azure Blob Storage"
last-updated: 2021-05-13
primary-nav: integrations
secondary-nav: non-fastly-services
tags: integrations

---

This is a test!!!

[Microsoft Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/) のパブリックコンテナとプライベートコンテナは、Fastly でオリジンとして使用できます。

{% alert tip %}適切に設定されたサービスにより、Fastly と Microsoft のお客様は Fastly と Azure の ExpressRoute Direct Local との統合のメリットを受けることができるほか、Azure からのアウトバウンドデータ転送コストを Fastly の標準価格に含めることができます。 詳細については、[Azure からのアウトバウンドデータ転送ガイド](/ja/guides/outbound-data-transfer-from-azure)をご覧ください。{% endalert %}

## Azure Blob Storage をオリジンとして使用する

Azure Blob Storage をオリジンとして使用する際、Azure ストレージサービスの最新バージョンのご利用を推奨します。古いすぎるバージョンや、リクエストヘッダーで指定されていないバージョンを使用する場合は、互換性の問題が発生する可能性があります。ヘッダーに指定されないバージョンの使用を強制する場合、Blob ストレージサービスにて [`DefaultServiceVersion`](https://docs.microsoft.com/en-us/rest/api/storageservices/versioning-for-the-azure-storage-services) を設定します。

または、VCL スニペットを `vcl_miss` と `vcl_pass` に配置して、バージョンを設定または強制することもできます。例えば、バージョンが `2020-06-12` の場合、VCL スニペットは次のようになります。

{% raw %}
```vcl
{{if (req.backend.is_origin) { }}
{{ set bereq.http.x-ms-version = "2020-06-12"; }}
}
{% endraw %}

Once you've configured your Azure Blob Storage stores are ready to make them available through Fastly, follow the steps below.

### Creating a new service

Follow the instructions for [creating a new service](/en/guides/working-with-services#creating-a-new-service). You'll add specific details about your origin when you fill out the **Create a new service** fields:

  * In the **Name** field, enter any descriptive name for your service.
  * In the **Domain** field, enter the hostname you want to use as the URL (e.g., `cdn.example.com`).
  * In the **Address** field, enter `<storage account name>.blob.core.windows.net`.
  * In the **Transport Layer Security (TLS)** area, leave the **Enable TLS?** default set to **Yes** to secure the connection between Fastly and your origin.
  * In the **Transport Layer Security (TLS)** area, enter `<storage account name>.blob.core.windows.net` in the **Certificate hostname** field.

### Setting the default Host and correct path

Once the new service is created, set the default Host to `azure` and then add your container path to the URL by following the steps below:

1.  {% t step-select-service %}
2.  {% t step-click-edit %}
3.  {% t step-click-settings-tab %}
4.  Click the **Override host** switch. The Override host header field appears.

    ![the Add an override host header window](/img/guides/override-host-azure-blob.png)

5.  Enter the hostname of your Azure Blob Storage account. For example, `<storage account name>.blob.core.windows.net`.

6.  Click the **Save** button. The new override host header appears in the Override host section.
7.  {% t step-click-content-tab %}
8.  Click the **Create header** button. The Create a header page appears.
9.  Fill out the **Create a header** fields as follows:
    *   In the **Name** field, enter `Modify URL`.
    *   From the **Type** menu, select **Request**, and from the **Action** menu, select **Set**.
    *   In the **Destination** field, enter `url`.
    *   In the **Source** field, enter `"/<your container name>" req.url`.
    *   From the **Ignore if set** menu, select **No**.
    *   In the **Priority** field, enter `10`.
10.  Click the **Create** button. The new Modify URL header appears on the Content page.
11.  {% t step-activate-deploy %}

### Testing your results

By default, we create DNS mapping called **yourdomain.global.prod.fastly.net**. In the example above, it would be `cdn.example.com.global.prod.fastly.net`. Create a DNS alias for the domain name you specified (e.g., CNAME `cdn.example.com` to `global-nossl.fastly.net`).

Fastly will cache any content without an explicit `Cache-Control` header for 1 hour. You can verify whether you are sending any cache headers using cURL. For example:

```plaintext
$ curl -I opscode-full-stack.blob.core.windows.net

HTTP/1.1 200 OK
Date: Fri, 04 May 2018 21:23:07 GMT
Content-Type: application/xml
Transfer-Encoding: chunked
Server: Blob Service Version 1.0 Microsoft-HTTPAPI/2.0
```

この例では、Cache-Control ヘッダーが設定されていないため、デフォルトの TTL が適用されます。

## Azure Blob Storage プライベートコンテナの使用

Azure Blob Storage のプライベートコンテナを Fastly で使用する場合には、次の指示に従ってください。

### ご利用の前に

[正しいコンテナを指定し](#azure-blob-storage-をオリジンとして使用する)、オリジンをポート443に設定することで、Azure Blob Storage コンテナが Fastly で利用可能になっていることを確認してください。これは認証前に行う必要があります。

設定を完了するには、Azure Blob Storage 認証ヘッダーを構築するために、Azure Storage アカウントの共有キーとストレージアカウント名が必要になります。認証ヘッダーは次のような形式になります。

```plaintext
Authorization: SharedKey `_Account name_`:`_Signature_`
```

最後に Blob Storage コンテナ名も必要になりますので、ご注意ください。

### 共有キーを使って Fastly が Azure Blob Storage プライベートコンテナを使用するように設定

{% alert warning %}
お客様のアカウントの共有キーでは、詳細なアクセス管理をすることができません。お客様の共有キーにアクセスすることで、誰でもコンテナを読み、書き込みすることができます。そのため、[共有アクセス署名](#shared-access-signature-sas-によって-azure-blob-storage-のプライベートコンテナを使用するように-fastly-を構成する) (SAS) の使用をお勧めします。
{% endalert %}

共有キーを使用して Fastly で Azure Blob Storage のプライベートコンテナにアクセスするには、Microsoft の[共有キーによる認証](https://docs.microsoft.com/en-us/rest/api/storageservices/authorize-with-shared-key)ページを参照してください。次に、[2つのヘ](ッダーを作成します/ja/guides/adding-or-modifying-headers-on-http-requests-and-responses)。日付ヘッダー (認証署名用) と認証ヘッダーを作成します。

#### 日付ヘッダーの作成

以下の手順で、日付ヘッダーを作成します。

1. {% t step-login %}
2. {% t step-select-service %}
3. {% t step-click-edit %}
4. {% t step-click-content-tab %}
5. **Create a header** ボタンをクリックします。Create a header ページが表示されます。

   ![ヘッダーページから日付ヘッダーを作成](/img/guides/create-new-date-header.png)

6. 各 **Create a header** フィールドに次の要領で入力します。
   * **Name** フィールドに `Date` と入力します。
   * **Type** メニューから **Request** を、**Action** メニューから** Set **を選択します。
   * **Destination** フィールドに、`http.Date` と入力します。
   * **Source** フィールドに `now` と入力します。
   * **Ignore if set** メニューから **No** を選択します。
   * **Priority** フィールドに `10` と入力します。
7. **Create** ボタンをクリックします。Content ページに新しい日付ヘッダーが表示されます。これは後で認証ヘッダーの署名で使用します。

#### 認証ヘッダーの作成

次に、以下の仕様で認証ヘッダーを作成します。

1. **Create header** ボタンを再度クリックして、別の新しいヘッダーを作成します。Create a header ページが表示されます。

   ![ヘッダーページから認証ヘッダーを作成する](/img/guides/create-authorization-header-azure-blob.png)

2. 各 **Create a header** フィールドに次の要領で入力します。
   * **Name** フィールドに `Azure Authorization` と入力します。
   * **Type** メニューから **Request** を、**Action** メニューから** Set **を選択します。
   * **Destination** フィールドに、`http.Authorization` と入力します。
   * **Ignore if set** メニューから **No** を選択します。
   * **Priority** フィールドに `20` と入力します。
3. **Source** 欄には、ヘッダー認証情報を次の形式で入力します。

   ```plaintext
   "SharedKey <Storage Account name>:" digest.hmac_sha256_base64(digest.base64_decode("<Azure Storage Account shared key>"), if(req.method == "HEAD", "GET", req.method) LF LF LF req.http.Date LF "/<Storage Account name>" req.url.path)
   ```

   `<Storage Account name>` と `<Azure Storage Account shared key>` は、お客様が事前に収集した情報に置き換えます。例：

   ```plaintext
   "SharedKey test123:" digest.hmac_sha256_base64(digest.base64_decode("UDJXUFN1NjhCZmw4OWo3MnZUK2JYWVpCN1NqbE93aFQ0d2hxdDI3"), if(req.method == "HEAD", "GET", req.method) LF LF LF req.http.Date LF "/test123" req.url.path)
   ```

   [Source フィールドに入力するパラメーター](#ソースフィールドの詳細を確認する)の詳細は、以下のとおりです。

4. **Create** ボタンをクリックします。新しい認証ヘッダーが Content ページに表示されます。
5. {% t step-activate-deploy %}

#### Source フィールド入力内容の詳細

認証ヘッダーの Source フィールドを詳しく見てみましょう。基本的な形式はこちらです。

`SharedKey<storage account name><Signature Function><key><message>`

上記のパラメーターは、次の情報を伝えています。

| **要素** | **説明** |
---------------------|------------------------------------------------------------------------------------------------------------------------------------------
| `SharedKey` | ストレージアカウント名の前に置かれる定数。これは常に SharedKey である必要があります。 |
| `storage account name` | Azure Storage アカウント名。この例では `test123` を使用しています。 |
| `signature function` | 署名のキーとメッセージを検証するためのアルゴリズム。この例では `digest.hmac_sha256_base64(<key>, <message>)` を使用しています。 |
| `key` | Azure Storage 開発者アカウントからの Azure Storage アカウント共有キー。この例では `UDJXUFN1NjhCZmw4OWo3MnZUK2JYWVpCN1NqbE93aFQ0d2hxdDI3` を使っています。このキーは、Base64 でデコードされている必要があります。 |
| `message` | UTF-8 エンコードされた StringToSign。メッセージの各部分の内訳は以下の表をご覧ください。 |

認証ヘッダーの Source フィールドに含まれるメッセージは、このような基本的な形式になっています。

`<HTTP-verb></n><Content-MD5>/n<Content-Type></n><Date></n><CanonicalizedAmzHeader></n><CanonicalizedResource>`

上記のパラメーターは、次の情報を伝えています。

| **要素** | **説明** |
-------------------------|---------------------------------------------------------------------------------------------------------------------------------
| `HTTP-verb` | REST 動詞。この例では `req.method` を使用しています。HEAD を GETに書き換えるのは、オリジンにリクエストを送る前に Varnish が内部的に行っているからです。 |
| `\n` | 改行表示の定数。この値は常に \n です。 |
| `Content-MD5` | メッセージの整合性チェックとして使用される content-md5 ヘッダー値。空欄であることが多いです。この例では、`LF` (ラインフィード) を使用しています。 |
| `Content-Type` | MIME-type を指定するための Content-type ヘッダーの値。空欄であることが多いです。この例では `LF` を使用しています。 |
| `Date` | 日付と時間のタイムスタンプ。上記の手順で最初に別のヘッダーとして作成した `req.http.Date` を使用しています。 |
| `CanonicalizedHeaders` | Azure Blob Storage の実装をカスタマイズする x-ms ヘッダー。空欄であることが多いです。この例では `LF` を使用しています。 |
| `CanonicalizedResource` | Storage アカウント名。この例では `"/test123"` を使用しています。 |

### 共有アクセス署名 (SAS) を使って Azure Blob Storage のプライベートコンテナの使用するように Fastly を設定する

サービス共有アクセス署名 (SAS) を利用して Fastly で Azure Blob Storage のプライベートコンテナにアクセスするには、[共有アクセス署名を使用したアクセスの委任](https://docs.microsoft.com/en-us/rest/api/storageservices/delegate-access-with-shared-access-signature)ページを参照してください。次に SAS を取得し、アクセスに署名します。

{% alert short: tip %}
サービス共有アクセス署名を使用することで、以下をより詳細にコントロールすることが可能です。　

* 開始時刻と終了時刻を含む、SAS の有効期間。
* SASによって許可された権限。例えば、SAS によって blob に対して読み取りと書き込みの権限が与えられても、削除の権限は与えないことが可能です。
* Azure Storage が SAS を受信する任意 IP アドレスや IP アドレスの範囲。例えば、お客様の組織に属する IP アドレスの範囲を指定することができます。
* Azure Storage が SAS を受信するプロトコル。このオプションパラメーターを使用して、HTTPS を利用するクライアントのみにアクセスを制限することができます。
   {% endalert %}

#### 共有アクセス署名の取得

以下の手順で SAS を取得します。

1. Azure ポータルで、Storage アカウントに移動します。
2. Settings メニューの **Shared access signature** に移動します。 Shared access signature メニューが表示されます。

   ![共有アクセス署名の作成](/img/guides/create-shared-access-signature-azure-blob.png)

3. **Allowed services** から、**Blob** を選択します。
4. **Allowed resource types **から、**Object** を選択します。
5. **Allowed permissions** から、**Read** を選択してください。
6. **Start time** は現在の日時のままにします。
7. **End time** をお好きな日時に設定してください (下記の注意をご参照ください)。
8. **Allowed protocols** を **HTTPS only** に設定します。
9. **Generate SAS and connection string** ボタンをクリックします。生成された情報が表示されます。
10. **SAS token** 欄のコンテンツをコピーして保存します。 SAS トークンは次のようなものになります。

   ```
   ?sv=2017-11-09&ss=b&srt=o&sp=r&se=2019-10-22T15:41:23Z&st=2018-10-22T07:41:23Z&spr=https&sig=decafbaddeadbeef
   ```

   [共有アクセス署名のパラメーター](#共有アクセス署名のパラメータの詳細を確認)の詳細については、以下で詳しく説明しています。

#### URL の署名

次に、以下の手順で認証ヘッダーを作成し、アクセス URL に署名します。

1. {% t step-login %}
2. {% t step-select-service %}
3. {% t step-click-edit %}
4. {% t step-click-content-tab %}
5. **Create a header** ボタンをクリックします。Create a header ページが表示されます。

   ![ヘッダーページからURLを変更する](/img/guides/modify-url-shared-access-signature-azure-blob.png)

6. 各 **Create a header** フィールドに次の要領で入力します。
   * **Name** 欄に、`Set Azure private SAS Authorization URL` など名前を入力します。
   * **Type** メニューから **Request** を、**Action** メニューから** Set **を選択します。
   * **Destination** フィールドに、`url` と入力します。
   * **Source** 欄に `req.url.path {"<SAS TOKEN>"}` を入力し、`{"<SAS TOKEN>"}` を Azure ポータルから得たトークンに置き換えます。
   * **Ignore if set** メニューから **No** を選択します。
   * **Priority** フィールドに `10` と入力します。
7. **Create** ボタンをクリックします。Content ページに新しいヘッダーが表示されます。
8. {% t step-activate-deploy %}

#### 共有アクセス署名パラメーターの詳細

Microsoft の[サービス SAS を作成する](https://docs.microsoft.com/en-us/rest/api/storageservices/create-service-sas)ページでは、共有アクセス署名の詳細とその構築方法について説明しています。

| **要素** | **説明** |
-------------------------|---------------------------------------------------------------------------------------------------------------------------------
| `sv` | `signedversion` のフィールド。これは必須であり、Azure ポータルが提供したものでなければなりません。 |
| `ss` | `signedservice` のフィールド。これは必須であり、「blog storage」の `b`でなければなりません。 |
| `srt` | `signedresourcetype` のフィールド。これは必須であり、「object」の `o`でなければなりません。 |
| `sp` | `signedpermissions` のフィールド。これは必須であり、「read only」の`r`でなければなりません |
| `st` | `signedstart` のフィールド。これはオプションです。共有アクセス署名が有効になる時刻を ISO 8601 と互換性のある UTC 形式で指定します。時刻が入力されない場合、このコールの開始時刻はストレージサービスがリクエストを受信した時刻と想定されます。 |
| `se` | `signedexpiry` のフィールド。必須項目です。共有アクセス署名が無効になる時刻を ISO 8601 と互換性のある UTC 形式で指定します。 |
| `spr` | `signedprotocol` のフィールド。オプションです。コンテナがアクセスに使用する HTTP プロトコル (`http` または `https`) を指定します。Fastly では `https` を推奨します。 |
| `sig` | `signature` のフィールド。これは必須であり、Azure ポータルが提供したものでなければなりません。 |

{% alert warning %}
`se` の有効期限は把握しておく必要があります。有効期限を過ぎてしまうと、Fastly がプライベートコンテナにアクセスできなくなります。
{% endalert %}

## パフォーマンスを向上させる TCP 接続の設定

Fastly のデフォルト設定では、パフォーマンス向上のために、オリジンに対して確立された TCP 接続を開いたままにします。しかし Azure では、アイドル状態の接続は切断されるようにデフォルト設定されています。つまり、フローのアイドルタイムアウト時間がデフォルトの4分に達すると、Azure ロードバランサーによってフローが停止されるようになっています。統合を確実に完了するため、Azure をチューニングする2つの方法をお勧めします。

* Azure ロードバランサーの[TCP アイドルタイムアウト設定](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-tcp-idle-timeout)にてタイムアウト値を増やす
* アイドルタイムアウト時に双方向の [TCP リセット](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-tcp-reset) (RST パケット) を送信するようにAzureを設定する

{% include dry/third-party-integrations.html %}
