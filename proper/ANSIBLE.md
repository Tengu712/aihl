# Ansible

## What's Ansible

IaCツールの一種。
リモートマシン上の構成・設定をコードで管理することを主な目的とする。
命令的な管理方法であり、どちらかというとCI/CDに近い。
SSHでユーザにログイン、sudoに昇格してファイルを更新したり・サービスを再起動したりする。
基本的にリモート側で行う環境構築はPython 3のインストールだけ。

Terraformと違ってインフラ自体の構成管理はしない。
あくまで、マシン上で行うコマンド実行や・設定ファイルの置換作業などを代替してくれるツール。
また、stateのようなsingle sourceがないため、過去の実行からの差分検知をしない。
つまり、サービスAを定義・適用した後、その定義をコードから削除・適用しても、サービスAは残存・稼働する。

https://docs.ansible.com

## Quick Setup

以下、すべてクライアントマシンでの操作:

1. Ansibleをインストール。macOSならHomebrewからもインストール可能。WindowsはWSL等必須。
2. IaCリポジトリに次の`inventory.ini`を作成。`HOSTNAME`はIP直打ちよりもSSH config上のhost nameにすると良い。

```
[myhosts]
HOSTNAME
```

3. `ansible myhosts -i inventory.ini -m ping`を実行して正常に動作するか確認

## Interactive Login

リモート側がSSHログインに対して対話的な認証を必要とする場合、コード適用に失敗する。
ので、一度SSHログインを通してから、そのセッションを利用する必要がある。
以下をSSH configに追記:

```
Host HOSTNAME
	HostName       HOSTURI
	User           USERNAME
	Port           PORT
	IdentityFile   KEYFILEPATH
	ControlMaster  auto
	ControlPath    ~/.ssh/cm-%C
	ControlPersist 1m
```

以下をansible.cfgに追記:

```
[ssh_connection]
ssh_args = -C
```

次のようにログイン、セッション利用:

```sh
ssh HOSTNAME true && ansible GROUP -i inventory.ini -m ping
```

## Command

次のように`ansible.builtin.command`でコマンドを実行できる:

```yml
ansible.builtin.command:
  cmd: "echo hoge > /home/foo/bar"
  creates: "/home/foo/bar"
become_user: USERNAME
```

`creates`にコマンド実行によって作成が期待されるファイル・ディレクトリを指定することで、それが存在しない場合に限りコマンドを実行するようにできる。
逆に言えば、`creates`の指定されていない場合は常に実行される。

> [!NOTE]
> コマンド実行は常に`~/.ansible/tmp`を一時ディレクトリとして使おうとする。
> そのため、ホームディレクトリを持たないユーザを`become_user`に指定すると`Unable to use '/nonexistent/.ansible/tmp' as temporary directory, falling back to system default.`のような警告が出る。
> 最終的に`/tmp`にフォールバックするので問題ないが、警告を消したい場合は`vars: ansible_remote_tmp: /tmp`を指定する。
