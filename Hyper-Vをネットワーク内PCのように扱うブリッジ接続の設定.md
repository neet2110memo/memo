# Hyper-Vをネットワーク内PCのように扱うブリッジ接続の設定

> 環境: Windows10 21H1 WSL有

> :warning: 定期的にネットワークが切れてしまう不具合があるようです。ハードの構成によって相性がある可能性があります。
>> この不具合は`Intel Ethernet Connection I217-V`で発生しましたが、ドライバを最新にしてから設定の`Receive-Side Scaling`を無効<sup>[3](https://answers.microsoft.com/ja-jp/windows/forum/windows_10-networking-winpc/windows-10/aa9f0579-fd4f-4112-ad7a-a06fe5598c65)</sup>しにて、さらにコマンドプロンプト管理者権限で`Receive-Side Scaling（RSS）の無効化`のコマンド`netsh int tcp set global rss=disabled`を設定<sup>[4](https://xtech.nikkei.com/it/article/COLUMN/20100824/351391/)</sup>すると改善しました。

## 概要

Hyper-Vネットワーク接続する場合デフォルトだと`Default Switch`, `外部ネットワーク`, `内部ネットワーク`, `プライベートネットワーク`とあります。<sup>[1](https://www.atmarkit.co.jp/ait/articles/2008/14/news018.html)</sup>  
`外部ネットワーク`で`管理オペレーションシステムにこのネットワークアダプターの共有を許可する`にするとインターネットにもつなげてかつ親機のプライベートネットワークに接続できるはず・・・ができなかったので解決方法として`ブリッジ接続`する方法を書きます。  
`ブリッジ接続`だと物理PCがハブに接続されている状態と同じようになるので扱い易いです。

## 方法

1．**仮想スイッチの作成**

- Hyper-Vの`仮想スイッチマネージャー`で`新しい仮想ネットワークスイッチ`から`内部`を選択して`仮想スイッチの作成`ボタンをクリックします。
- 名前はここでは`Internal for Hyper-V`ということにします。

2．**ネットワークブリッジの作成**

- `コントロール パネル\ネットワークとインターネット\ネットワーク接続`を開きます。
- インターネットに接続している名前(ここでは`イーサネット`)と先ほど作成した`vEthernet (Internal for Hyper-V)`の二つを同時選択します。
- 右クリックして`ブリッジ接続`をクリックします。
- （おそらくここでエラーが起きます。選択した２つのうちどちらかがブリッジ接続から外れてしまっていますので右クリックから`ブリッジへ追加`をクリックします。）
- `ネットワーク ブリッジ`が作成されます。

3．**DHCPでIPを割り振る**

> :bulb: 固定IPにする場合は通常の固定方法で固定を割り振ります。この行程は必要ありません。

- `ネットワーク ブリッジ`がDHCPより割り振れれないためインターネットにつなげられない状態となります。
- DHCP割り振りにする場合には`ForceCompatibilityMode`をEnableにする必要があります。
- コマンドプロンプトを管理者で起動します。

- 次のコマンドを実行します。

  ```bat: 管理者: コマンドプロンプト
  netsh bridge show adapter
  ```

  結果

  ```bat: 管理者: コマンドプロンプト
  ----------------------------------------------------------------------
  ID AdapterFriendlyName         ForceCompatibilityMode
  ----------------------------------------------------------------------
    1 vEthernet (Internal for Hyper-V) disabled
    2 イーサネット                      disabled
  ----------------------------------------------------------------------
  ```

- disableになっているのでenableにする必要があります。
  次のコマンドを実行します。`netsh bridge set adapter {アダプタの番号} e`

  ```bat: 管理者: コマンドプロンプト
  netsh bridge set adapter 1 e
  netsh bridge set adapter 2 e
  ```

- 次のコマンドを実行します。

  ```bat: 管理者: コマンドプロンプト
  netsh bridge show adapter
  ```

  結果

  ```bat: 管理者: コマンドプロンプト
  ----------------------------------------------------------------------
  ID AdapterFriendlyName         ForceCompatibilityMode
  ----------------------------------------------------------------------
    1 vEthernet (Internal for Hyper-V) enabled
    2 イーサネット                      enabled
  ----------------------------------------------------------------------
  ```

- enableになっていることを確認して終了です。

## 参考URL

1. [仮想スイッチの種別と用途 Windows 10 Hyper-V入門](<https://www.atmarkit.co.jp/ait/articles/2008/14/news018.html>)
2. [WindowsPCとLinuxサーバーをブリッジ接続する方法](<https://qiita.com/nkojima/items/4056d749328d4512c53f>)
  の`4. ForceCompatibilityModeの設定`
3. [Windows 10 でイーサネット接続エラーが表示されてインターネットに接続できなくなる。](https://answers.microsoft.com/ja-jp/windows/forum/windows_10-networking-winpc/windows-10/aa9f0579-fd4f-4112-ad7a-a06fe5598c65)
4. [［Windows 7編］ネットワーク設定を標準で使ってはいけない](<https://xtech.nikkei.com/it/article/COLUMN/20100824/351391/>)
