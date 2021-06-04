# AWS Amazon Web Service 設定

## Amazon Linux 2にAWS CLIをインストール

Dockerの`AmazonLinux2`に`AWS CLI ver.2`<sup>[2](#ref2)</sup>をインストールします。

パッケージ`less`をインストールします。

```bash: bash
yum install less
```

インストールファイルをダウンロードします。

```bash: bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

本物であることを確認します。

```bash: bash
gpg --keyserver "hkp://keys.gnupg.net" --recv "A6310ACC4672"
curl -o "awscliv2.sig" "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip.sig"
gpg --verify "awscliv2.sig" "awscliv2.zip"
```

解凍してインストールします。

```bash: bash
unzip "awscliv2.zip"
"./aws/install"
rm -rf "awscliv2.zip"
```

動作を確認します。

```bash: bash
aws --version
```

`Dockerfile`に記述する場合は連続して書けます。

```Dockerfile: AWS CLI ver.2
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && \
    gpg --keyserver "hkp://keys.gnupg.net" --recv "A6310ACC4672" && \
    curl -o "awscliv2.sig" "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip.sig" && \
    gpg --verify "awscliv2.sig" "awscliv2.zip" && \
    unzip "awscliv2.zip" && \
    "./aws/install" && \
    rm -rf "awscliv2.zip" && \
    aws --version
```

## WindowsのPowoerShellにAWSPowerShellをインストール

> 環境: Windows 10 Pro 21H1, PowerShell 7.1.3  

下記より`AWSPowerShell`<sup>[1](#ref1)</sup>モジュールをインストールします。  

PowerShell(管理者)

```ps1: install
Install-Module -Name AWSPowerShell
Set-ExecutionPolicy RemoteSigned
```

> :bulb: 以下の2つは不要なのでインストールする必要はありません。  
> `AWS.Tools.Common`  
> `AWSPowerShell.NetCore`  

`AWSPowerShell`モジュールをインポートして、リージョンとアクセスキーIDを入力します。インポートはPowerShellを起動するたびに毎回行う必要があります。

```ps1: ログイン
Import-Module AWSPowerShell
Set-DefaultAWSRegion ap-northeast-1
Set-AWSCredential -AccessKey AXXXXXXXXXXXXXXXXXXX
```

アクセスキーIDを入力して実行した後にシークレットキーを聞かれるので入力します。

```ps1: Set-AWSCredential
cmdlet Set-AWSCredential at command pipeline position 1
Supply values for the following parameters:
SecretKey:
```

ログインします。

```ps1: ログイン
(Get-ECRLoginCommand).Password | docker login --username AWS --password-stdin xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/repo-name
```

:bulb: 認証情報を保存することもできます。

保存

```ps1:
Set-AWSCredential -AccessKey {key} -SecretKey {secretKey} -StoreAs {name}
```

読み込み

```ps1:
Set-AWSCredentials -ProfileName {name}
```

保存ファイルの場所

```bat: RegisteredAccounts.json
%LOCALAPPDATA%\AWSToolkit\RegisteredAccounts.json
```

暗号化されて保存されます。  
RegisteredAccounts.json

```json: RegisteredAccounts.json
{
    "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" : {
        "AWSAccessKey" : "31B465964896E87EF0894C85B792CE9AA2C5F7D73AA660D9F80559218632CC60000000000E8000000002000020000000DAFE50D95B2D4BEAB7DBFD80DA69C7DA44D475E61D37C30DC779CDA007F9E575300000006E777CE74CACD1FEF8C40CFE6911126EBDBA9DD269FAF7F1FC32E414F70E9BA77AF6A1CFBD90386FF026CD2FF3A81C8340000000958AE42820729C6335155A76381D7D91C4F0AFE1B4B5E8BB1E6D4A5C36C8C41669BF9DEA6FFEBB57CBC01A902A3994AA5D5E433E8F787278B8843A710C759080",
        "AWSSecretKey" : "4B68F8BD736CB0F3D2BF89684125114B72F0EED745937970C43108FAFB7D95DE000000000E8000000002000020000000912F1A46A1288EF2B16AEBFEF7D27E8E8D36E47511EBCCB1EB7B18E8E3F8B4CE10000000690CFECACCD99D4A18257C497CB9092340000000A9B886192ADBF3634C49F43489F44E351F467C052510CE0683FBFFAE06E594A3D20F68D0F404F8A9559952D21B25745F1B294E67501E8C699B51B11FFF90FFEB",
        "ProfileType"  : "AWS",
        "DisplayName"  : "{name}"
    }
}
```

## AWS ECRにDockerイメージをアップロード（プッシュ）する方法

> 環境: Windows 10 Pro 21H1, PowerShell 7.1.3  

Fargateで使用するためのDockerイメージをAmazon Elastic Container Registry (ECR)にプッシュします。  
IAMで作成したユーザーのアクセスキーIDとシークレットキーは既にあるものとします。

1. **アップロードするDockerイメージの作成**
    ローカルPCのDockerで作成してください。

2. **作成したdockerイメージの`IMAGE ID`の確認**
    例: `5f4598ddb811`

3. **レポジトリの作成**
    アップロード先のレポジトリを作成します。
    [Amazon Container Services > ECR](https://ap-northeast-1.console.aws.amazon.com/ecr/repositories?region=ap-northeast-1)で`レポジトリの作成`ボタンをクリックします。レポジトリ名はローカルのDockerイメージと合わせる必要はありません。ここでは`repo-name`ということにします。

4. **レポジトリURIの取得**
    3で作成したレポジトリのURIを取得します。
    例: `xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/repo-name`

5. **レポジトリURIのタグ付け**
    作成したイメージにタグ（別のイメージ名）を付けます。

    ```bat: Docker tag
    docker tag 5f4598ddb811 xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/repo-name
    ```

6. **AWS認証**
    本ページの`WindowsのPowoerShellにAWSPowerShellをインストール`の項目を参照してください。

7. **レポジトリにプッシュ**

    作成したレポジトリのURIに`Docker push`します。

    ```ps1: アップロード
    docker push xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/repo-name:latest
    ```

    アップロードの様子

    ```ps1: アップロード
    The push refers to repository [xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/repo-name]
    5f70bf18a086: Pushed
    8f9729464ed2: Pushed
    4354f8c2d895: Pushing [==============================================>    ]  205.6MB/222.2MB
    3afa7e0a726a: Pushed
    e9badb1bcd18: Pushing [================================>                  ]  183.3MB/284.7MB
    a7079a8d1ba4: Pushing [===================>                               ]  151.5MB/385.1MB
    aa3e0fd29ca1: Pushing [==============================>                    ]    184MB/298.6MB
    22b0b18ae9c3: Pushing [==========================================>        ]  139.8MB/163.2MB
    ```

    :warning: アップロードに結構時間がかかります。アップロード帯域が太い環境でも3時間以上かかるかもしれません。何らかの処理をしながらアップロードしているようです。

## AWS CloudWatchのイベントを追加する

> 環境: [Docker Amazon Linux 2](https://hub.docker.com/_/amazonlinux/)

`AmazonLinux2`に`AWS CLI ver.2`<sup>[2](#ref2)</sup>がインストールしてある前提です。

1. **ポリシー設定**
    `IAM > アクセス管理 > ポリシー`から以下の項目のアクセス権限のあるポリシーを作成して、使用するユーザーにポリシーをアタッチします。
    - EventBridge : PutRule
    - EventBridge : DeleteRule
    - EventBridge : PutTargets
    - IAM : PassRole
    もしくは、（面倒くさい場合）既存の`CloudWatchEventsFullAccess`をアタッチします。

2. **ログイン情報の設定**
    環境変数に登録するとログインできるようになります。

    ```bash: bash
    export AWS_DEFAULT_REGION=ap-northeast-1
    export AWS_ACCESS_KEY_ID=AXXXXXXXXXXXXXXXXXXX
    export AWS_SECRET_ACCESS_KEY=XXX
    ```

3. **イベントの操作**
    ルールを作成します。
    cron式は`分 時 日 月 曜日`<sup>[3](#ref3)</sup>となります。次の例の10回のトリガーは以下となります。  
    Cron 式 `0 10 * * ? *`
    - Fri, 04 Jun 2021 10:00:00 GMT
    - Sat, 05 Jun 2021 10:00:00 GMT
    - Sun, 06 Jun 2021 10:00:00 GMT
    - Mon, 07 Jun 2021 10:00:00 GMT
    - Tue, 08 Jun 2021 10:00:00 GMT
    - Wed, 09 Jun 2021 10:00:00 GMT
    - Thu, 10 Jun 2021 10:00:00 GMT
    - Fri, 11 Jun 2021 10:00:00 GMT
    - Sat, 12 Jun 2021 10:00:00 GMT
    - Sun, 13 Jun 2021 10:00:00 GMT

    今回は分をランダムにしたいので`$(( $(od -vAn -N4 -tu4 < /dev/random) % 60 ))`として0-59になるようにしています。

    ```bash: bash
    minute=$(( $(od -vAn -N4 -tu4 < /dev/random) % 60 ))
    aws events put-rule --name {event name} --schedule-expression "cron($minute 10 * * ? *)" --state ENABLED
    ```

    > **:bulb:ヒント** ルールを変更するにはもう一度同じルール名で`aws events put-rule`すれば上書きされます。この時ターゲットはそのままになります。  

    ルールに追加するターゲットのjsonファイルを作成します。ここではイベント発生時に`ECS タスク`としてFargateのタスクを実行する例を示します。jsonファイルは現在の作業ディレクトリに配置します。  
    scheduledtask.json

    ```json: scheduledtask.json
    [{
        "Id": "{task-id}（イベントルール内での任意のターゲット名）",
        "Arn": "arn:aws:ecs:ap-northeast-1:xxxxxxxxxxxx:cluster/{cluster-name}",
        "RoleArn": "arn:aws:iam::xxxxxxxxxxxx:role/ecsTaskExecutionRole",
        "EcsParameters": {
                "TaskDefinitionArn": "arn:aws:ecs:ap-northeast-1:xxxxxxxxxxxx:task-definition/{task-name}:{リビジョン番号}（無記入ならlatest）",
                "TaskCount": 1,
                "LaunchType": "FARGATE",
                "NetworkConfiguration": {
                        "awsvpcConfiguration": {
                                "Subnets": ["subnet-xxxxxxxx"],
                                "SecurityGroups": ["sg-xxxxxxxxxxxxxxxxx"],
                                "AssignPublicIp": "ENABLED"
                        }
                },
                "PlatformVersion": "LATEST"
        }
    }]
    ```

    ルールにターゲットを追加します。

    ```bash: bash
    aws events put-targets --rule {event name} --targets file://scheduledtask.json
    ```

    イベントを削除するコマンドは下記です。

    ```bash: bash
    aws events delete-rule --name {event name}
    ```

## 参考資料

1. [Windows PowerShell 用 AWS Tools (モジュール) を見つける](https://www.powershellgallery.com/packages/AWS.Tools.Common/)<a name="ref1"></a>
2. [Linux での AWS CLI バージョン 2 のインストール、更新、およびアンインストール](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/install-cliv2-linux.html)<a name="ref2"></a>
3. [Cron Format](http://www.nncron.ru/help/EN/working/cron-format.htm)<a name="ref3"></a>
