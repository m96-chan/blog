---
title: 'Root済みAndroidにUbuntu 24.04をchroot構築する完全ガイド【Linux Deploy不要】'
description: 'Linux Deployを使わず、Termux + chroot でUbuntu 24.04をAndroid上に構築。SSHサーバー化して画面OFFでも24/7運用する方法。ハマりポイント全部書いた。'
category: 'Tech'
pubDate: '2026-02-15'
heroImage: '../../assets/broken-android-hero.jpg'
---

# Linux Deployはもう要らない

Root済みAndroidにLinux環境を入れるとき、定番は**Linux Deploy**だ。GUIでポチポチ、ディストロ選んでインストール。簡単。

...のはずが、実際には：

- **Archを入れたらSSHが起動しない**（バグ）
- Ubuntu 18が入る（古すぎてNode.js動かない）
- 謎のエラーが頻発

結局、**Termux + 手動chroot が最強**だった。

この記事では、Root済みAndroidにUbuntu 24.04 (Noble Numbat) をchroot環境として構築し、SSHサーバーとして24/7運用するまでの手順を解説する。

## 前提条件

- Root済みAndroid端末（Magisk推奨）
- Termux インストール済み（[F-Droid](https://f-droid.org)か[GitHub Releases](https://github.com/termux/termux-app/releases)から）
- ストレージに最低2GB以上の空き

今回使った端末は **Nothing Phone (1)**。Snapdragon 778G+、arm64。

## 1. rootfsを取得する

Termuxを開いて、root権限に入る：

```bash
pkg install tsu
su
```

Ubuntu Base のrootfs tarball をダウンロード：

```bash
cd /data/local
mkdir -p ubuntu
curl -kLO https://cdimage.ubuntu.com/ubuntu-base/releases/noble/release/ubuntu-base-24.04.4-base-arm64.tar.gz
tar -xzf ubuntu-base-24.04.4-base-arm64.tar.gz -C ubuntu
```

> **注意**: URLのバージョンは変わる。[公式ページ](https://cdimage.ubuntu.com/ubuntu-base/releases/noble/release/)で最新を確認。`24.04.2`は存在しない（ハマった）。

## 2. chroot環境をマウントする

```bash
mount --bind /dev /data/local/ubuntu/dev
mount -t devpts devpts /data/local/ubuntu/dev/pts
mount --bind /proc /data/local/ubuntu/proc
mount --bind /sys /data/local/ubuntu/sys
```

chrootに入る：

```bash
chroot /data/local/ubuntu /bin/bash
```

入ったら最初にやること：

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export TMPDIR=/tmp
export HOME=/root
mkdir -p /tmp
```

> **⚠️ TMPDIRの罠**: TermuxのTMPDIR（`/data/data/com.termux/files/usr/tmp/`）がchroot内に漏れる。`export TMPDIR=/tmp` しないと、`ca-certificates`のインストールなどで `mktemp: failed to create file via template` エラーが出る。

## 3. DNS・ホスト設定

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "127.0.0.1 localhost" > /etc/hosts
```

## 4. Androidネットワークグループの設定

**これをやらないとaptがダウンロードできない。** Android chroot特有の罠。

```bash
groupadd -g 3003 aid_inet
groupadd -g 3004 aid_net_raw
groupadd -g 1003 aid_graphics
usermod -g 3003 -G 3003,3004 -a _apt
usermod -G 3003 -a root
```

`_apt`ユーザーにAndroidのネットワーク関連グループ（`aid_inet`, `aid_net_raw`）を追加しないと、aptのダウンロードプロセスがネットワークアクセスできない。

## 5. aptリポジトリの設定

Ubuntu 24.04からaptのソース定義が**DEB822形式**に変わった。

```bash
mkdir -p /etc/apt/sources.list.d
```

`/etc/apt/sources.list.d/ubuntu.sources` を以下の内容で作成：

```
Types: deb
URIs: http://ports.ubuntu.com/ubuntu-ports
Suites: noble noble-updates
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

> **注意**: arm64は`ports.ubuntu.com/ubuntu-ports`を使う。x86の`archive.ubuntu.com`ではない。

echoで書く場合（viが無いので）：

```bash
echo 'Types: deb' > /etc/apt/sources.list.d/ubuntu.sources
echo 'URIs: http://ports.ubuntu.com/ubuntu-ports' >> /etc/apt/sources.list.d/ubuntu.sources
echo 'Suites: noble noble-updates' >> /etc/apt/sources.list.d/ubuntu.sources
echo 'Components: main restricted universe multiverse' >> /etc/apt/sources.list.d/ubuntu.sources
echo 'Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg' >> /etc/apt/sources.list.d/ubuntu.sources
```

> **⚠️ ファイルが書けてない罠**: `wc -l` で確認すること。0行なら書き込み失敗してる。`mkdir -p` でディレクトリを先に作ってるか確認。

古いキャッシュが残ってるとbackportsやsecurityのエラーが出るので：

```bash
rm -rf /var/lib/apt/lists/*
apt update
```

20万パッケージくらい取得されれば成功。50個とかなら`Components`が欠けてる。

## 6. 基本パッケージのインストール

Ubuntu Baseはマジで何も入ってない。`cat`すら無い。

```bash
apt install -y coreutils openssh-server vim curl wget
```

## 7. SSHサーバーの設定

```bash
passwd root
mkdir -p /run/sshd
sed -i 's/#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
/usr/sbin/sshd -p 2222
```

ポート22はAndroid側と競合する可能性があるので、2222を使う。

PCから接続テスト：

```bash
ssh root@<端末のIP> -p 2222
```

## 8. .profileの設定

毎回exportするのは面倒なので：

```bash
echo 'export TMPDIR=/tmp' >> /root/.profile
echo 'export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin' >> /root/.profile
echo 'export HOME=/root' >> /root/.profile
```

## 9. 自動起動（Magisk service.d）

再起動するとchroot環境は消える。Magiskの`service.d`にスクリプトを置いて自動起動させる。

**chroot外**（Termuxのsu状態）で：

```bash
cat > /data/adb/service.d/chroot-ubuntu.sh << 'SCRIPT'
#!/system/bin/sh
sleep 10
mount --bind /dev /data/local/ubuntu/dev
mount -t devpts devpts /data/local/ubuntu/dev/pts
mount --bind /proc /data/local/ubuntu/proc
mount --bind /sys /data/local/ubuntu/sys
chroot /data/local/ubuntu /bin/bash -c "export TMPDIR=/tmp; export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin; mkdir -p /run/sshd; /usr/sbin/sshd -p 2222"
SCRIPT
chmod +x /data/adb/service.d/chroot-ubuntu.sh
```

これでAndroid起動時に自動でchroot + SSHサーバーが立ち上がる。画面OFFでもOK。

## ハマりポイントまとめ

今回踏んだ地雷を全部列挙する。未来の自分と読者のために。

| 問題 | 原因 | 解決 |
|------|------|------|
| Linux Deploy ArchでSSH起動しない | アプリのバグ | Linux Deploy捨ててTermux chroot |
| pacman Landlock エラー | カーネルが古い | `--disable-sandbox` |
| pacman alpmユーザー不在 | 最小構成 | `/etc/passwd`に手動追加 |
| Node.js動かない | Ubuntu 18のglibc古い | Ubuntu 24.04に入れ直し |
| tarball URL 404/503 | バージョン違い | 公式ページで最新確認 |
| apt "not signed" | backports/securityキャッシュ残り | `rm -rf /var/lib/apt/lists/*` |
| apt パッケージ50個しかない | sourcesファイル書き込み失敗 | `wc -l`で確認、`mkdir -p`忘れ |
| `cat`が無い | Ubuntu Baseは超最小構成 | `apt install coreutils` |
| ca-certificates インストール失敗 | TermuxのTMPDIRが漏れてる | `export TMPDIR=/tmp` |
| SSH Connection refused | ポート競合 or mount不足 | ポート変更 + mount確認 |

## まとめ

Linux Deployを使わず、Termux + chroot でUbuntu 24.04環境を構築した。結果的に**最小構成で最も軽量**な環境になった。

rootさえあれば、余計なアプリもデーモンも不要。`chroot` + `sshd` だけ。Nothing Phone (1) がarm64サーバーとして静かに動いてる。

使い古しのAndroidがサーバーになる時代、Mac Miniを並べてる場合じゃない。
