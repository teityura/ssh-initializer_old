# ssh-initializer

VMに初回接続するときなどに、

1. ~/.ssh/config 書いて、
2. ssh-copy-id | authorized_keysの追記 して、
3. sshで rootログインできるようにして、
4. 一般ユーザで パスなしsudoできるようにして、
5. パスワード変更する

という、諸々の作業をコマンド1発でできるようにしました

## 概要

> roles/ssh/

group_vars/all.yml に記載した内容をもとに、  
.ssh/ 以下に config, id_rsa, rsa.pub を生成します

> roles/users/

hosts.ini, group_vars/all.yml に定義した ホスト, ユーザ に対し、下記を実行します

- authorized_keys を追記 (add_authorized_key.yml)
- パスなしsudo を許可 (allow_sudo.yml)
- sshの rootログインを許可 (allow_root_login.yml)
- パスワードを変更 (set_password.yml)

## 使い方

``` txt
vim hosts.ini
vim group_vars/all.yml
ansible -u ubuntu -m ping

ansible-playbook site.yml
ansible -u root -m ping
```

``` ini:hosts.ini
[kube:children]
master
worker

[master]
kubem1 ansible_password=hogem1
kubem2 ansible_password=hogem2
kubem3 ansible_password=hogem3

[worker]
kubew1 ansible_password=hogew1
kubew2 ansible_password=hogew2
kubew3 ansible_password=hogew3
```

``` yml:group_vars/all.yml
INITIALIZE_USER: ubuntu

USERS:
  - ubuntu

PASSWORD: "hogehoge"

GROUP_NAME: kube

GROUP_PARAMS:
  - { name: kubem1, ip: 10.10.10.11 }
  - { name: kubem2, ip: 10.10.10.12 }
  - { name: kubem3, ip: 10.10.10.13 }
  - { name: kubew1, ip: 10.10.10.21 }
  - { name: kubew2, ip: 10.10.10.22 }
  - { name: kubew3, ip: 10.10.10.23 }
```

## hosts.ini について

host.ini には ホストを定義しておきます  

ホスト名 は なんでもよく、  
グループ は あってもなくてもよいです

## group_vars/all.yml について

> INITIALIZE_USER

初回接続するときのユーザを指定します

> USERS

パスなしsudoをしたいユーザを羅列します

> PASSWORD

変更後のパスワードを指定します

> GROUP_PARAMS

.ssh/config の生成に利用し、下記のような設定ができます

``` txt
Host kubem1
  Hostname 10.10.10.11
  User root
  IdentityFile {{ playbook_dir }}/.ssh/{{ GROUP_NAME }}.id_rsa
```

(そのため ip には ホスト名(Hostname) も指定できます)

### パスワード認証するとき (recommended)

ansible_password にパスワードを入れてください  
(ホストごとにパスワードが違っていても、実行可能です)

### 鍵認証するとき (deprecated)

.ssh/{{ GROUP_NAME }}.id_rsa を設置してください  

ホストごとに鍵が違う場合、  
同じ鍵でログインできるように手動でauthorized_keysを追記してください  
(動作未確認です)

#### ホストに初回接続するときの挙動について

ホストによってパスワードが違う場合、  
hosts.ini で ansible_password を定義しておくと、  
別々のパスワードでログインさせることができます

#### ホストに2回目以降接続するときの挙動について

1回目でプレイブックが実行できていたら、  
rootユーザー の authorized_keys が登録された状態になるので、  
ansible_password が変更前の値になっていても、  
鍵認証でログインできるようになっています
