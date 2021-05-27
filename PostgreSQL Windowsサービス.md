# PostgreSQLをWindows10の標準ユーザーのサービスとして運用する

Linuxのように管理者以外でPostgreSQLサービスを立ち上げようとすると意外と手間取るのでその設定方法を記録しました。

- 普通にインストールすると管理者としての運用となります。
- 標準ユーザーでサービス起動するときの設定の仕方を説明します。

## データフォルダの配置

1. **PostgreSQLをインストールする**  
    ここでは`postgresql-13.3-2-windows-x64`をインストールしました。
    とりあえずデフォルトのままインストールします。

2. **OSのユーザー作成**  
    まずWindowsの標準ユーザーを作成します。ここでは例として`postgres`というユーザー名とします。

3. **データフォルダの作成**  
    OSの`postgres`ユーザーとしてログインします。  
    データの場所は`D:\psql\13\data`に配置するとして作成します。  
    dataフォルダのセキュリティで`アクセス許可`に次の2つのユーザーを追加します。

    |ユーザー|アクセス許可|
    |:-:|:-:|
    |postgres|フルコントロール|
    |NETWORK SERVICE|フルコントロール|

    （権限変更がうまくいかないことがあったのでここで一旦ログアウトしてログインし直した方がよいかもしれないです。）

4. **データベースを初期化して作成する**  
    initdbで初期データベースを作成します。
    コマンドプロンプトで下記を実行します。PostgreSQLのユーザー名は`postgres`とします。

    ```cmd:コマンドプロンプト
    "C:\Program Files\PostgreSQL\13\bin\initdb.exe" -D "D:\psql\13\data" --username=postgres --encoding=UTF-8 --locale=C --auth=scram-sha-256 --pwprompt
    ```

    （日本語ロケールは遅くなるだけという情報があるので--locale=Cとして無効にしています。）  
    実行結果は下記のようになります。これは手動でコマンドプロンプトでPostgreSQLを起動する場合のコマンドとなります。サービス登録しない場合にはこれで起動させます。

    ```cmd:結果
    ^"C^:^/Program^ Files^/PostgreSQL^/13^/bin^/pg^_ctl^" -D ^"D^:^\psql^\13^\data^" -l ログファイル start
    ```

## ファイアウォールの設定

プラグラムおよびサービスで下記を指定します。  
`%ProgramFiles%\PostgreSQL\13\bin\postgres.exe`

## サービスの登録

- **サービスの実体**  
    レジストリの中にサービスの実体があります。デフォルトインストールのPostgreSQLのサービスはレジストリエディタで下記を開くと見れます。  
    `コンピューター\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\postgresql-x64-13`  
    サービスを新しく登録するにはsc.exeを実行してレジストリに追加します。
- **サービスの削除**  
    デフォルトでインストールした時のサービス`postgresql-x64-13`を削除します。

    ```cmd:サービスの削除
    sc delete "postgresql-x64-13"
    ```

- **サービスの新規作成**  
    ここではサービス名を`psql-x64-13`を例として登録します。OS起動時に自動でサービス立ち上げする場合は`start= auto`にします。手動なら`start= demand`、無効なら`start= disable`とします。

    ```cmd:サービスの登録
    sc create "psql-x64-13" binPath= """"C:\Program Files\PostgreSQL\13\bin\pg_ctl.exe""" runservice -N """psql-x64-13""" -D """"D:\psql\13\data""" -w" depend= "RPCSS" DisplayName= "psql-x64-13 - PostgreSQL Server 13" error= normal obj= "NT AUTHORITY\NetworkService" type= own start= auto
    ```

- **サービスの説明文の追加**  
    新規作成時に説明文を追加できないので必要であれば後から追加する。

    ```cmd:サービスの説明文追加
    sc description "psql-x64-13" "Provides relational database storage."
    ```

- **サービスの開始、終了**  
    scで起動すると非同期、netで起動すると同期となります。スクリプトなどで起動するときは失敗を検知するためにnetの方が便利です。

    ```cmd:サービス開始1
    sc start "psql-x64-13"
    ```

    ```cmd:サービス開始2
    net start "psql-x64-13"
    ```

    > :bulb: **ヒント**  
    サービス開始に失敗するときは、データフォルダのアクセス権限に`NetworkService`がフルコントロールとなっているか確認してください。

- **サービスの実行権限を変更する**  
    これが標準ユーザーでサービス登録するときの肝です。まず、作成したサービスの`アクセス許可`を下記コマンドで取得します。

    ```cmd:アクセス許可を取得
    sc sdshow "psql-x64-13"
    ```

    実行結果

    ```cmd:結果
    D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWLOCRRC;;;IU)(A;;CCLCSWLOCRRC;;;SU)
    ```

    これにサービスを実行するユーザーのセキュリティ識別子(SID)を追加します。`NETWORK SERVICE`のSIDは`S-1-5-20`と決まっていますので先ほどの実行結果に`(A;;RPWP;;;S-1-5-20)`を追加します。下記のコマンドで追加した`アクセス許可`を上書きします。

    ```cmd:SID上書き
    sc sdset "psql-x64-13"  D:(A;;RPWP;;;S-1-5-20)(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWLOCRRC;;;IU)(A;;CCLCSWLOCRRC;;;SU)
    ```

    実行結果

    ```cmd:結果
    [SC] SetServiceObjectSecurity SUCCESS
    ```

    > :bulb: `NETWORK SERVICE`などの既知のSIDではない場合は`whoami /<user>`コマンドでSIDを調べます。  
    "RPWP"は"RP(Start)とWP(Stop)の操作を許可"という意味のようです。

## **レプリケーションをスクリプトから行うための設定**  

冗長化スクリプトを使用するための個人的な設定メモです。

- DBにレプリケーションに必要な設定を加える  
    レプリケーションを実行するユーザー名を`repl`とした時のレプリケーションに必要な権限などを設定します。  
    まず、psqlコンソールにログインします。

    ```cmd:psql設定
    "C:\Program Files\PostgreSQL\13\bin\psql.exe" -U postgres -h localhost -p 5432 -d postgres
    ```

    psqlコンソールで以下のコマンドを実行して設定をします。

    ```psql:replに関する設定
    SET password_encryption = 'scram-sha-256';
    CREATE ROLE repl WITH REPLICATION LOGIN;
    CREATE DATABASE replication WITH OWNER = 'repl';
    GRANT EXECUTE ON function pg_catalog.pg_ls_dir(text, boolean, boolean) TO repl;
    GRANT EXECUTE ON function pg_catalog.pg_stat_file(text, boolean) TO repl;
    GRANT EXECUTE ON function pg_catalog.pg_read_binary_file(text) TO repl;
    GRANT EXECUTE ON function pg_catalog.pg_read_binary_file(text, bigint, bigint, boolean) TO repl;
    GRANT EXECUTE ON function pg_promote TO repl;
    GRANT EXECUTE ON function pg_ls_waldir TO repl;    
    ```

    新しく作成した`repl`ユーザーにパスワードを設定します。

    ```psql:パスワード設定
    \password repl
    ```

## **postgresql.confの設定**  

`postgresql.conf`ファイルはデータフォルダ内にあります。  
アーカイブフォルダを`C:\Users\postgres\psql\13\archivedir`、
ログフォルダを`C:\Users\postgres\psql\13\log`
とした時の設定例です。  

```Properties:postgresql.confの変更箇所
archive_mode = on
archive_command = 'copy "%p" "C:\\Users\\postgres\\psql\\13\\archivedir\\%f"'
restore_command = 'copy "C:\\Users\\postgres\\psql\\13\\archivedir\\%f" "%p"'
max_wal_senders = 10
wal_keep_size = 1024
max_replication_slots = 10
hot_standby = on
log_destination = 'stderr'
logging_collector = on
log_directory = 'C:\\Users\\postgres\\psql\\13\\log'
log_replication_commands = on
```

`C:\Users\postgres\psql`フォルダの`アクセス許可`は以下のようにしてください。

|ユーザー|アクセス許可|
|:-:|:-:|
|postgres|フルコントロール|
|NETWORK SERVICE|読み取り|

## **pg_hba.confの設定**  

`pg_hba.conf`ファイルはデータフォルダ内にあります。（HBA とは、host-based authentication: ホストベース認証の略です）

```Properties:pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     scram-sha-256
host    all             all             samenet                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     scram-sha-256
host    replication     all             samenet                 scram-sha-256
```

## パスワードを書いたファイルを保管する場合

Linux版での`~/.pgpass`ファイルはWindows版だと
`%APPDATA%\postgresql\pgpass.conf`となります。  
書式は`hostname:port:database:username:password`のように書きます。

```txt:pgpass.conf
localhost:5432:postgres:postgres:postgrespassword
localhost:5432:replication:repl:replpassword
psqlsrv1:5432:postgres:postgres:postgrespassword
psqlsrv1:5432:replication:repl:replpassword
psqlsrv2:5432:postgres:postgres:postgrespassword
psqlsrv2:5432:replication:repl:replpassword
```

`pgpass.conf`の権限は下記のようにします。

|ユーザー|アクセス許可|
|:-:|:-:|
|postgres|フルコントロール|
|NETWORK SERVICE|読み取り|

<br>
