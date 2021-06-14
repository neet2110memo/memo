# AWS Amazon Web Service 設定

## Amazon Linux 2にAWS CLIをインストール

Dockerの`AmazonLinux2`に`AWS CLI ver.2`<sup>[2](#ref2)</sup>をインストールします。

パッケージ`less`をインストールします。

```bash
yum install less
```

インストールファイルをダウンロードします。

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

本物であることを確認します。

```bash
gpg --keyserver "hkp://keys.gnupg.net" --recv "A6310ACC4672"
curl -o "awscliv2.sig" "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip.sig"
gpg --verify "awscliv2.sig" "awscliv2.zip"
```

解凍してインストールします。

```bash
unzip "awscliv2.zip"
"./aws/install"
rm -rf "awscliv2.zip"
```

動作を確認します。

```bash
aws --version
```

`Dockerfile`に記述する場合は連続して書けます。

```Dockerfile
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

```powershell
Install-Module -Name AWSPowerShell
Set-ExecutionPolicy RemoteSigned
```

> :bulb: 以下の2つは不要なのでインストールする必要はありません。  
> `AWS.Tools.Common`  
> `AWSPowerShell.NetCore`  

`AWSPowerShell`モジュールをインポートして、リージョンとアクセスキーIDを入力します。インポートはPowerShellを起動するたびに毎回行う必要があります。

```PowerShell
Import-Module AWSPowerShell
Set-DefaultAWSRegion ap-northeast-1
Set-AWSCredential -AccessKey AXXXXXXXXXXXXXXXXXXX
```

アクセスキーIDを入力して実行した後にシークレットキーを聞かれるので入力します。

```PowerShell
cmdlet Set-AWSCredential at command pipeline position 1
Supply values for the following parameters:
SecretKey:
```

ECRのレポジトリにログインする例

```PowerShell
(Get-ECRLoginCommand).Password | docker login --username AWS --password-stdin xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/repo-name
```

:bulb: 認証情報を保存することもできます。

保存

```PowerShell
Set-AWSCredential -AccessKey {key} -SecretKey {secretKey} -StoreAs {name}
```

読み込み

```PowerShell
Set-AWSCredentials -ProfileName {name}
```

保存ファイルの場所

```batch
%LOCALAPPDATA%\AWSToolkit\RegisteredAccounts.json
```

暗号化されて保存されます。  
RegisteredAccounts.json

```json
{
    "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" : {
        "AWSAccessKey" : "3AA660D9F80559218632CC60DAFE50D95B2D4BEAB7DBFD80DA69C7DA44D475E61D37C30DC779CDA007F9E575300000006E777CE74CACD1FEF8C40CFE6911126EBDBA9DD269FAF7F1FC32E414F70E9BA77AF6A1CFBD90386FF026CD2FF3A81C8340000000958AE42820729C6335155A76381D7D91C4F0AFE1B4B5E8BB1E6D4A5C",
        "AWSSecretKey" : "ED745937970C43108FAFB7D95DE912F1A46A1288EF2B16AEBFEF7D27E8E8D36E47511EBCCB1EB7B18E8E3F8B4CE10000000690CFECACCD99D4A18257C497CB9092340000000A9B886192ADBF3634C49F43489F44E351F467C052510CE0683FBFFAE06E594A3D20F68D0F404F8A9559952D21B25745F1B294E67501E8C699B51B",
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

    ```batch
    docker tag 5f4598ddb811 xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/repo-name
    ```

6. **AWS認証**
    本ページの`WindowsのPowoerShellにAWSPowerShellをインストール`の項目を参照してください。

7. **レポジトリにプッシュ**
    作成したレポジトリのURIにレポジトリにログインします。

    ```PowerShell
    (Get-ECRLoginCommand).Password | docker login --username AWS --password-stdin xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/repo-name
    ```

    作成したレポジトリのURIに`Docker push`します。

    ```PowerShell
    docker push xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/repo-name:latest
    ```

    アップロードの様子

    ```PowerShell
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

    ```bash
    export AWS_DEFAULT_REGION=ap-northeast-1
    export AWS_ACCESS_KEY_ID=AXXXXXXXXXXXXXXXXXXX
    export AWS_SECRET_ACCESS_KEY=XXX
    ```

3. **イベントの操作**
    ルールを作成します。
    cron式は`分 時 日 月 曜日 年`<sup>[3](#ref3)</sup>となります。次の例の10回のトリガーは以下となります。  
    Cron 式 `0 10 * * ? *`

    ```log
    Fri, 04 Jun 2021 10:00:00 GMT
    Sat, 05 Jun 2021 10:00:00 GMT
    Sun, 06 Jun 2021 10:00:00 GMT
    Mon, 07 Jun 2021 10:00:00 GMT
    Tue, 08 Jun 2021 10:00:00 GMT
    Wed, 09 Jun 2021 10:00:00 GMT
    Thu, 10 Jun 2021 10:00:00 GMT
    Fri, 11 Jun 2021 10:00:00 GMT
    Sat, 12 Jun 2021 10:00:00 GMT
    Sun, 13 Jun 2021 10:00:00 GMT
    ```

    今回は分をランダムにしたいので`$(( $(od -vAn -N4 -tu4 < /dev/random) % 60 ))`として0-59になるようにしています。bashで一般的な`$((​$RANDOM % 60))`はAmazonLinux2ではできませんでした。

    ```bash
    minute=$(( $(od -vAn -N4 -tu4 < /dev/random) % 60 ))
    aws events put-rule --name {event name} --schedule-expression "cron($minute 10 * * ? *)" --state ENABLED
    ```

    > **:bulb:ヒント** ルールを変更するにはもう一度同じルール名で`aws events put-rule`すれば上書きされます。この時ターゲットはそのままになります。  

    ルールに追加するターゲットのjsonファイルを作成します。ここではイベント発生時に`ECS タスク`としてFargateのタスクを実行する例を示します。jsonファイルは現在の作業ディレクトリに配置します。  
    scheduledtask.json

    ```json
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

    ```bash
    aws events put-targets --rule ${event name} --targets file://scheduledtask.json
    ```

    イベントを削除するコマンドは下記です。

    ```bash
    aws events delete-rule --name ${event name}
    ```

## 参考資料

1. [Windows PowerShell 用 AWS Tools (モジュール) を見つける](https://www.powershellgallery.com/packages/AWS.Tools.Common/)<a name="ref1"></a>
2. [Linux での AWS CLI バージョン 2 のインストール、更新、およびアンインストール](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/install-cliv2-linux.html)<a name="ref2"></a>
3. [Cron Format](http://www.nncron.ru/help/EN/working/cron-format.htm)<a name="ref3"></a>

## ECSタスク

### タスク定義の作成

`新しいタスク定義の作成か`ら作成します。
`FARGATE`を選択して`次のステップ`をクリックします。  
ここで必要な事項

- タスク定義名
- タスクロール（必要であれば）
- タスクメモリ
- タスクCPU
- コンテナの定義
  - コンテナの追加
  ここではコマンドや環境変数はしていしなくても、タスクの実行時に指定できます。

タスク定義を変更する場合は、変更するタスクを選んで`新しいリビジョンの作成`をクリックします。

### タスク定義の削除

`アクション > 登録解除`だと消えないで`Inactive`となります。2021年時点では完全な削除はできなく、リビジョンが無限に上がっていく仕様のようです。

### タスクの実行

タスクの実行はクラスターから行います。

- 起動タイプ
    `FARGATE`を選択します。
    タスクごとにキャパシティープロバイダー戦略を変更できます。
    `FARGATE`または`FARGATE_SPOT`の割合を決めます。
- タスク定義
    タスク定義名とリビジョンを選択します。
- クラスター
    クラスター名を選択します。
- クラスターVPC
    `vpc-b81cf3de (172.31.0.0/16)`を選択します。
    これは最初に実行したときに自動定義されたものだと思われる？
- サブネット
    デフォルトで`ap-northeast-1a/1c/1d`の3つが選択できます。（複数可）
- セキュリティグループ
    ポートフォワーディングの許可を設定するための項目です。デフォルトだと自動で`HTTP(TCP Port80)`を解放した新しいセキュリティグループが作成されます。毎回増えてしまうので同じものなら名前を付けてそれを選びます。（TODO: 受付ない場合はポート解放なしでOK？）
- パブリックIPの自動割り当て
    `ENABLED`を選択しないと外部に出れない？（TODO: 要調査）
- 詳細オプション
  - タスクの上書き
    - タスクロール
        なし
    - タスク実行ロール
        なし
  - コンテナの上書き
    - コマンドの上書き
        コンテナ実行時に渡したい引数を指定します。dockerの`CMD`の設定です。
    - 環境変数の上書き
        渡したい環境変数を指定します。`docker run -e {環境変数名}={環境変数値}`の設定です。
    - 環境ファイルの上書き
        `S3`にあるファイルを指定します。環境変数は`docker run --env-file {環境ファイル名.env}`の設定です。

### イベントルール作成時の`ECSタスク`設定の項目

ブラウザからの設定では設定項目が限られているようです。

- クラスター
- タスク定義
- 起動タイプ
- タスク定義リビジョンの設定
- サブネット
- セキュリティグループ
- 自動割り当てパブリックIP
- この特定のリソースに対して新しいロールを作成する

コンテナの上書きができないためイベントルールごとに`CMD`を変更するのに不便です。
`AWS CLI`からだとjsonファイルで指定するので詳細な設定ができます。  
[参考コード](https://qiita.com/hiko1129/items/d87323a4268513bb7a62)  
[Add container overrides to put-targets for ECS tasks #2760](https://github.com/aws/aws-cli/issues/2760)

```bash
aws events put-rule --name "${evnet_rule_name}" --schedule-expression "${cron_format}" --state ENABLED
aws events put-targets \
    --rule "${evnet_rule_name}" \
    --targets="[
        {
            \"Id\": \"${evnet_rule_name}\",
            \"Arn\": \"arn:aws:ecs:ap-northeast-1:xxxxxxxxxxxx:cluster/${cluster_name}\",
            \"RoleArn\": \"arn:aws:iam::xxxxxxxxxxxx:role/${role_name}\",
            \"Input\": \"{\\\"containerOverrides\\\":[{\\\"name\\\":\\\"${container_name}\\\",\\\"command\\\":[\\\"${cmd_arg1}\\\",\\\"${cmd_arg2}\\\"],\\\"entryPoint\\\":[\\\"${ep_arg1}\\\",\\\"${ep_arg2}\\\"],\\\"workingDirectory\\\":\\\"${working_directory}\\\"}]}\",
            \"EcsParameters\": {
                \"TaskDefinitionArn\": \"arn:aws:ecs:ap-northeast-1:xxxxxxxxxxxx:task-definition/${task_name}\",
                \"TaskCount\": 1,
                \"LaunchType\": \"FARGATE\",
                \"NetworkConfiguration\": {
                    \"awsvpcConfiguration\": {
                        \"Subnets\": [
                            \"ap-northeast-1a\"
                            \"ap-northeast-1c\",
                            \"ap-northeast-1d\",
                        ],
                        \"SecurityGroups\": [
                            \"sg-xxxxxxxxxxxxxxxxx\"
                        ],
                        \"AssignPublicIp\": \"ENABLED\"
                    }
                },
                \"PlatformVersion\": \"LATEST\"
            }
        }
    ]"
```

Inputの中身は適宜変更して書き込みます。ただし--targetsに入れるときに`"`は`\\\"`とするのと`改行`は削除してください。InputなしでもOKです。  

JSONフォーマットのファイルから読み込む場合は以下ようにして置き換えてセットします。

target.json

```json
{
    "Id": "${evnet_rule_id_name}",
    "Arn": "arn:aws:ecs:ap-northeast-1:xxxxxxxxxxxx:cluster/${cluster_name}",
    "RoleArn": "arn:aws:iam::xxxxxxxxxxxx:role/${role_name}",
    "Input": "${input_json}",
    "EcsParameters": {
        "TaskDefinitionArn": "arn:aws:ecs:ap-northeast-1:xxxxxxxxxxxx:task-definition/${task_name}",
        "TaskCount": 1,
        "LaunchType": "FARGATE",
        "NetworkConfiguration": {
            "awsvpcConfiguration": {
                "Subnets": [
                    "ap-northeast-1d",
                    "ap-northeast-1c",
                    "ap-northeast-1a"
                ],
                "SecurityGroups": [
                    "sg-xxxxxxxxxxxxxxxxx"
                ],
                "AssignPublicIp": "ENABLED"
            }
        },
        "PlatformVersion": "LATEST"
    }
}
```

input.json

```json
{
    "containerOverrides": [
        {
            "name": "${container_name}",
            "command": [
                "${cmd_arg1}",
                "${cmd_arg2}"
            ],
            "entryPoint": [
                "${ep_arg1}",
                "${ep_arg2}"
            ],
            "workingDirectory": "${working_directory}"
        }
    ]
}
```

```bash
# ルールを作成する
aws events put-rule \
  --name "${evnet_rule_name}" \
  --schedule-expression "${cron_format}" \
  --state ENABLED
# input.json
input_json=$(<input.json)
input_json="$(echo -e "$input_json" | tr -d '[:space:]')"
input_json="$(echo -e "$input_json" | tr -d '\n')"
input_json=${input_json//\"/\\\"}
# target.json
target_json=$(<target.json)
target_json=${target_json/"#{input_json}"/$input_json}
target_json=${target_json/"#{target_id}"/"my_target_id"}
echo $target_json
# JSON文字列をターゲットにセットして登録する。複数のターゲットを登録できる。
aws events put-targets \
  --rule "${evnet_rule_name}" \
  --targets="[$target_json]"
```

こちら↓の方法がもっと万能かもしれないです。
[ECS(Fargate)のバッチをCloudFormationで作成する](https://qiita.com/yktakaha4/items/f5a32d5f2e7a6263faa4)
