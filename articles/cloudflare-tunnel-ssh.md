---
title: 'Cloudflare Tunnelを使ってSSHする'
emoji: '☁️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['cloudflare', 'ssh']
published: false
---

つい先日 Raspberry Pi 4 Model B (8GB) を購入し、別のマシンや外出先からリモートで SSH できるようにしようと思っていたところ Cloudflare Zero Trust の Cloudflare Tunnel というものを知人から知ったので使ってみることにした。

# 前提条件

- ネームサーバーを Cloudflare にしていること

# Tunnel を作る

著者はダッシュボードを使う方法を使用したので、そちらだけ記事に書かせていただきます。
https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/remote/#set-up-a-tunnel-remotely-dashboard-setup

コマンドラインだけで設定するには公式のドキュメントを読んでみてください。
https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/local/#set-up-a-tunnel-locally-cli-setup

1. Cloudflare Zero Trust のダッシュボードへアクセスする
1. **Access > Tunnels**へ移動して、**Create a tunnel**をクリック
1. **Tunnel name**に名前を入力
1. 使っている OS とアーキテクチャを選択して、手順に従い`cloudflared`をサービス化する
1. **Route tunnel**でドメインを設定し、**Service**を**SSH**、URL に**localhost:22**を設定

## アプリケーションを追加

1. Cloudflare Zero Trust のダッシュボードへアクセスする
1. **Access > Application**へ移動して、**Add an application**をクリックし**Self-hosted**を選択
1. **Application domain**には**Route tunnel**に設定したドメインと同じものを設定
1. **Next**をクリックして、ポリシーやその他の設定を終えたら**Add application**をクリックして完了

:::message
**Browser rendering**を**SSH**に設定すると、ブラウザから SSH できるようになります。
:::

:::message alert
2022 年 9 月 7 日現在、**Cookie settings**にて**Enable Binding Cookie**を ON にすると`ssh`コマンドで接続できなくなる原因となるため、**SSH in Browser**しかしないという場合以外は ON にしないこと
:::

# SSH で接続する

## サーバー側

Cloudflare Tunnel を使うと**SSH ポートを開放する必要がない**ので、ファイヤーウォールの設定の設定とかは特に必要ありません。

`root`でログインさせないとか、パスワード認証使えないようにするとか最低限の設定だけしておけば大丈夫

### Browser Rendering を使う場合

**Short-lived certificates**の公開鍵を登録しておかないと毎回開くたびに公開鍵をよこせと要求されてしまうので、`sshd_config`に`TrustedUserCA`として設定しておきましょう。

まずは公開鍵を作成して、入手します。

- Cloudflare Zero Trust のダッシュボードへ移動
- **Access > Service Auth > SSH**の順で移動
- 作成したアプリケーションを選択して、**Generate certificate**をクリック
- 公開鍵をコピーして、`/etc/ssh/ca.pub`に内容を貼り付けて作成

`sshd_config`には以下のように設定を追加する

```diff:sshd_config
+ PubkeyAuthentication yes
+ TrustedUserCAKeys /etc/ssh/ca.pub
```

## クライアント側

`ssh-copy-id`等などの手段を用いてサーバーに公開鍵を登録しておきましょう。

**Short-lived certificates**を用いれば鍵作って登録して～みたいなことすらせずに済むのですが、何故かうまくいかなかったので著者は自分で作った鍵を登録する形にしました。

https://developers.cloudflare.com/cloudflare-one/identity/users/short-lived-certificates

### `~/.ssh/config`

```config:~/.ssh/config
Host ssh.example.com
  User inkohx
  IdentityFile 秘密鍵のパス
  ProxyCommand cloudflared access ssh --hostname %h
```

https://developers.cloudflare.com/cloudflare-one/tutorials/ssh/#native-terminal

:::message
ご存知かもしれませんが、ドキュメントに載っている方法だと使っているマシンのユーザーネームでアクセスしてしまうので`User`は設定したほうが良いでしょう。
:::

### 接続

`ssh ssh.example.com`

# 終わり

SSH in Browser できるのスマホからも操作できるしすごいけど、デスクトップからだとコピー＆ペーストをショートカットキーでできないの地味に不便だったりする。
