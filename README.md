(Japanese)

# Pacemaker STONITH検証環境用 playbook

このリポジトリは、 仮想マシンの電源断を IPMI経由で制御できるようにする VirtualBMC の設定を行う playbook です。

仮想環境上の Pacemakerクラスタの試験において、物理環境用のSTONITHプラグイン(fence_ipmilan, external/ipmi)設定をそのまま使用して設定・動作確認を行うことが可能になります。

あくまでPacemakerクラスタの検証環境用のツールですので、実商用環境での利用には向いていません。

VirtualBMCは、仮想環境のホストOS上にインストールして利用します。


## VirtualBMC について

VirtualBMCは、仮想マシンの電源を IPMI経由で制御できるようにする仮想IPMIサーバです。OpenStack プロジェクトの一部として開発されています。

このplaybookでインストールする VirtualBMC はここからフォークして一部独自拡張を行ったものです。

* VirtualBMC ホームページ(オリジナル)
  * https://docs.openstack.org/virtualbmc/latest/
  * https://github.com/openstack/virtualbmc

* 独自拡張版 VirtualBMC
  * https://github.com/linux-ha-japan/virtualbmc-vbox
  * VirtualBox (VBoxManage) 対応: [devel-vbox-2.0ブランチ](https://github.com/linux-ha-japan/virtualbmc-vbox/tree/devel-vbox-2.0)

* 留意点
  * 独自拡張はいずれも実験的実装です。本格的な利用には向いていません。
  * Pacemakerクラスタの検証環境での設定確認、デモレベルでの利用を想定しています。

## 前提

* ホストOS (動作確認済み環境)
  * Rocky 8 / RHEL 8
  * Windows 11 + WSL2 (Debian 20.04 LTS / AlmaLinux-9)
  * macOS 13.5(Ventura)
* 仮想環境
  * VirtualBox 7.1.12
  * KVM / libvirt (Linuxのみ)
* ホスト側OSに必要なパッケージ
  * git
  * ansible-core 2.14.18
* ホスト側OSに必要なパッケージ(KVM / libvirt の場合)
  * 開発ツール(Development Tools)
* 以下のコマンドは全て ```root``` で実行すること。

## VirtualBMC のインストール、設定

* (0) ホストOSにはあらかじめ git, ansible をインストールしておくこと。またlibvirt環境の場合は開発ツール(Development Tools)をインストールしておくこと。
```
# yum install git
# yum install ansible-core
```

* (1) playbook リポジトリの clone
```
# git clone https://github.com/linux-ha-japan/ansible-virtualbmc
# cd ansible-virtualbmc
```

* (2) 設定ファイルの作成: サンプルファイルを参考に下記の設定ファイル作成する。
  * ```hosts```
    * ドメイン名 (virsh / VBoxManage で表示されるゲスト名)
    * 仮想IPMI設定 (IPアドレス、ユーザ、パスワード)
  * ```group_vars/all.yml```
    * ```VBMC_VERSION```: libvirt / VirtualBox に合わせて指定
    * ```VBMC_IPMI_IF```: 仮想IPMI用IPアドレスを割り当てるホスト側のNIC
  * (```ansible.cfg```)
    * 省略可。好みに応じて設定してよい。

* (3) virtualbmc のインストール
```
# ansible-playbook -i hosts 10-vbmc-install.yml
```

## VirtualBMC の起動

* (1) vbmc(仮想IPMIデーモン)の起動
```
# ansible-playbook -i hosts 20-vbmc-start.yml 
```

* (2) vbmc が稼働していることを確認する。(Status が running)
```
# ./venv-vbmc/bin/vbmc list
+-------------+---------+----------------+------+
| Domain name | Status  | Address        | Port |
+-------------+---------+----------------+------+
| centos81-1  | running | 192.168.122.95 |  623 |
| centos81-2  | running | 192.168.122.96 |  623 |
+-------------+---------+----------------+------+
# 
```

* (3) ゲスト上のipmitoolで電源状態取得が可能なことを確認する。
```
[root@centos81-1 ~]# ipmitool -I lanplus -H 192.168.122.96 -U pacemaker -P pacemakerpass1 power status
Chassis Power is on

```
* (4) ゲスト上のipmitoolで対向ノードの電源断が可能なことを確認する。
```
[root@centos81-1 ~]# ipmitool -I lanplus -H 192.168.122.96 -U pacemaker -P pacemakerpass1 power off
Chassis Power Control: Down/Off

```

## 補足

* ホストOSを再起動した後は 20-vbmc-start.yml を再度実行して手動で起動すること(自動起動は未対応)。
* VirtualBMCのコマンド(vbmc)は ./venc-vbmc ディレクトリ配下に Python の virtualenv としてインストールされる。
* 手作業で設定を行う場合は ./venv-vbmc/bin/vbmc コマンドで可能。詳細はhelp等参照
* vbmc の設定ファイルは $HOME/.vbmc 配下に保存されている。
* vbmc の動作ログは /var/log/virtualbmc.log に出力される。
  * ログローテーションは設定されていないので注意。ただし通常はログはほとんど出ない。電源on/off動作のログを確認するにはdebugログの有効化が必要
