# Vagrant + CentOS + Nginx + Laravel + PHP + MySQL環境構築

| 使用技術    | バージョン    |
|:-----------|------------:|
| PHP        | 7.3         |
| Laravel    | 6.0         |
| MySQL      | 5.7         |
| OS         | CentOS7     |
| WebServer  | Nginx       |

#### 使用IP 192.168.33.19

<br>

## Vagrant仮想環境構築

```
$ vagrant init centos/7
```

centos/7のboxを指定し、vagrantを初期化
ディレクトリ内に仮想仮想マシンの初期設定ファイルであるVagrantfileの作成をする。

### Vagrantfileの編集を行います。
以下2行をコメントアウト
```
config.vm.network "forwarded_port", guest: 80, host: 8080
```
```
config.vm.network "private_network", ip: "192.168.33.10"
``` 
1行目ではポートフォワーディングの設定を行っています。
ゲストOSの80番ポートの通信をホストOSの8080番ポートへ転送する設定です。
2行目ではシステム内部での通信であるプライベートネットワークの設定を記述します。
<br>

今回使用するipは192.168.33.19であるため、２箇所目のコメントアウトを修正
```
config.vm.network "private_network", ip: "192.168.33.19"
```
<br>
```
config.vm.synced_folder "../data", "/vagrant_data"
```
以下に変更
```
config.vm.synced_folder "./", "/vagrant", type:"virtualbox"
```
ホストOSのディレクトリとゲストOSの/vagrantのディレクトリをリアルタイムで同期するための記述になります。
"./"はカレントディレクトリであるvirtual_env_manualにあたります。

<br>

### Vagrantプラグインのインストール
ここではVagrant-vbguestとsaharaというプラグインのインストールを行います。

#### vagrant-vbguest 
&emsp; GuestAdditionsのバージョンをvirtualBoxに合わせ最新化してくれるプラグイン。
#### sahara 
&emsp; 環境構築中のゲストOSの状態をサンドボックスという形で保存ができ、適宜巻き戻しができる。
```
$ vagrant plugin install vagrant-vbguest
$ vagrant plugin sahara
```
```
$ vagrant plugin list
```
プラグインがインストールされているか確認します。
```
sahara (0.0.17, global)
vagrant-vbguest (0.29.0, global)
```
このように表示されていればインストール完了です。

<br>

### vagrantを使用し、ゲストOSを起動します。
```
$ vagrant up
```

#### エラー発生
```
Vagrant cannot forward the specified ports on this VM, since they
would collide with some other application that is already listening
on these ports. The forwarded port to 8080 is already in use
on the host machine.

To fix this, modify your current project's Vagrantfile to use another
port. Example, where '1234' would be replaced by a unique host port:
```
8080ポートが既に使用されているとの内容
<br>
コマンドでポートを確認
```
$ lsof -i:8080 
COMMAND     PID    USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
com.docke 46735 asairyo   98u  IPv6 0x1781a46d18b75b79      0t0  TCP *:http-alt (LISTEN)
```
`kill 46735`を使用し、強制終了。
<br>

```
$ vagrant upでゲストOSを起動
[default] No Virtualbox Guest Additions installation found.
```
    
ちなみに以下のコマンドでvagrantのシャットダウンを行うことができます。
<br>
シャットダウン後はvagrant upでゲストOSの立ち上げができます。
```
$ vagrant halt
```

<br>

#### エラー発生
```
==> default: Checking for guest additions in VM...
default: No guest additions were detected on the base box for this VM! Guest
default: additions are required for forwarded ports, shared folders, host only
default: networking, and more. If SSH fails on this machine, please install
default: the guest additions and repackage the box to continue.
default: 
default: This is not an error message; everything may continue to work properly,
default: in which case you may ignore this message.
The following SSH command responded with a non-zero exit status.
Vagrant assumes that this means the command failed!

umount /mnt

Stdout from the command:

Stderr from the command:

umount: /mnt: not mounted
```

エラー内容からGuestAdditionがインストールされていないようなので、コマンドで確認します。
```
$ vagrant vbguest --status
[default] No Virtualbox Guest Additions installation found.
```
カーネルのアップデートを行います。
```
$ vagrant ssh
```
ゲストOSにログイン後以下のコマンドでアップデートを行います。
```
$ sudo yum -y install kernel
```
```
$ exit;
```
Vagrantの再起動を行います。
```
$ vagrant reload --provision
```

```
$ vagrant vbguest --status
[default] GuestAdditions 6.0.14 running --- OK.
```
インストールが完了し、動作も問題ないようです。

<br>

### ゲストOSが起動されるとログインが可能になります。
sshコマンドで確認。
```
$ vagrant ssh
[vagrant@localhost ~]$  
```
ログインに成功しました。

<br>

### sahara 
せっかくなのでpluginの項目でインストールしたsaharaを使ってみましょう

まずは状態を確認
ホストOSで
```
$ vagrant sandbox status
[default] Sandbox mode is off
```
Sandboxがoffになっていては使えないのでonで有効化しましょう
<br>
```
$ vagrant sandbox on
```
```
$ vagrant sandbox status
[default] Sandbox mode is on
```
有効化されました。
もう一度offにしたい場合は以下コマンドで無効化できます。

```
$ vagrant sandbox off
```
現時点での環境構築を保存し、いつでも元に戻せるようにサンドボックスをコミットします。
```
$ vagrant sandbox commit
[default] Committing the virtual machine...
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
```
これでコミット完了です。
```
vagrant sandbox rollback
```
上のコマンドを使えばいつでも元の状態に戻すことができます。


<br>

## パッケージのインストール
### 開発ツールのインストール
ゲストOS内部に開発に必要なパッケージを一括でインストールします。
```
[vagrant@localhost ~]$ sudo yum -y groupinstall "development tools"
```

#### sudo コマンド
&emsp; rootユーザー（管理者）の権限を借りるコマンド

#### yum 
&emsp; centosなどで使用されているパッケージ管理ツール・コマンドです。

#### groupinstall
&emsp; まとめてパッケージのインストールを行うことができるコマンドです

### PHPのインストール

```
[vagrant@localhost ~]$ sudo yum -y install epel-release wget
[vagrant@localhost ~]$ sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
[vagrant@localhost ~]$ sudo rpm -Uvh remi-release-7.rpm
[vagrant@localhost ~]$ sudo yum -y install --enablerepo=remi-php73 php php-pdo php-mysqlnd php-mbstring [vagrant@localhost ~]$ php-xml php-fpm php-common php-devel php-mysql unzip
```
php本体 関連モジュールをインストールします。
<br>
コマンドを実行し、インストールができたか確認しましょう。
```
[vagrant@localhost ~]$ php -v
PHP 7.3.26 (cli) (built: Jan  5 2021 10:36:07) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.3.26, Copyright (c) 1998-2018 Zend Technologies
```
<br>
phpのインストールが完了しました。

<br>

### Composerのインストール
```
[vagrant@localhost ~]$ php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
[vagrant@localhost ~]$ php composer-setup.php
[vagrant@localhost ~]$ php -r "unlink('composer-setup.php');"
[vagrant@localhost ~]$ sudo mv composer.phar /usr/local/bin/composer
[vagrant@localhost ~]$ composer -v
```
```
    ______
    / ____/___  ____ ___  ____  ____  ________  _____
    / /   / __ \/ __ `__ \/ __ \/ __ \/ ___/ _ \/ ___/
    / /___/ /_/ / / / / / / /_/ / /_/ (__  )  __/ /
    \____/\____/_/ /_/ /_/ .___/\____/____/\___/_/
                        /_/
    Composer version 2.0.9 2021-01-27 16:09:27
```
インストールが確認できました。

<br>

### MySQLのインストール
```
[vagrant@localhost ~]$ sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
[vagrant@localhost ~]$ sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm
[vagrant@localhost ~]$ sudo yum install -y mysql-community-server
```
```
[vagrant@localhost ~]$ mysql --version
mysql  Ver 14.14 Distrib 5.7.33, for Linux (x86_64) using  EditLine wrapper
```
無事インストールできました。
    
#### MySQLを起動し、接続を行います。
```
[vagrant@localhost ~]$ sudo systemctl start mysqld
[vagrant@localhost ~]$ mysql -u root -p
```
デフォルトでrootにパスワードが設定されているため、MySQLにアクセスできません。
一度パスワードを調べ、接続しパスワードの再設定が必要になります。

```
[vagrant@localhost ~]$ sudo cat /var/log/mysqld.log | grep 'temporary password'
```
コマンド実行後に以下のようにパスワードが表示されます。
```
2021-01-30T13:10:16.851002Z 1 [Note] A temporary password is generated for root@localhost: .iuoCw2fyz?i
```
このパスワードを入力し、MySQLへ接続します。
以下のコマンドでパスワードの設定が可能です。
```
mysql > set password = "新たなpassword";
```
しかし、MySQL5.7のパスワードポリシーが厳格なこともあり、開発段階では非常に手間です。
シンプルなパスワードで設定できるよう変更します。
```
[vagrant@localhost ~]$ sudo vi /etc/my.cnf
```
```
省略
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
#この1行を追記

```
編集後にMySQLの再起動を行います。
```
[vagrant@localhost ~]$ sudo systemctl restart mysqld
```
再起動したら先ほどのパスワードを入力し、MySQLにアクセスしましょう
<br>
以下のコマンドを実行してパスワードを設定したら完了です。
```
 mysql > set password = "新たなpassword";
```
<br>

## サーバーのインストール
### Nginxのインストール
ゲストOSにログインし、以下のコマンドを実行します。
```
[vagrant@localhost ~]$ sudo vi /etc/yum.repos.d/nginx.repo
```
ファイルに以下の内容を書き込みます。
```
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
gpgcheck=0
enabled=1
```
Nginxのインストールを行います。
```
[vagrant@localhost ~]$ sudo yum install -y nginx
```
バージョンを確認しましょう
```
nginx -v
nginx version: nginx/1.19.6
```
確認のため、Nginxを起動します。
```
[vagrant@localhost ~]$ sudo systemctl start nginx
```
```
[vagrant@localhost ~]$ sudo systemctl status nginx
```
上記コマンド起動を確認します。
問題なく起動できているのが確認できます。
<br>
192.168.33.19へアクセスすると
Welcome to nginx!の画面が表示されます。

<br>

## Laravelプロジェクト作成
ホストOSでプロジェクトを作成しましょう
```
$ composer create-project --prefer-dist laravel/laravel laravel_test "6.*"
```
```
$ cd laravel_test
```
```
$ php artisan serve
```
Laravelの画面が表示されれば成功です。

<br>

### MySQLでデータベースの作成、設定
LaravelでMySQLを使用するためにデータベースの作成、設定を行います。
```
$ mysql -u root -p
```
MySQLにログインし、以下コマンドを実行し、データベースを作成します。
```
create database laravel_test;
```
```
show databases;
```
作成したlaravel_testが表示されれば問題ありません。

Laravel側でDBを使用するための記述をします。
<br>
Laravelに使用するデータベースを教えてあげる必要があるため、laravel_app直下の.envファイルに記述をします。
```
.env
DB_DATABASE=laravel_test
DB_USERNAME=root
DB_PASSWORD="MySQLに設定しているパスワード"
```
`laravel_test/config/database.php`内のDB_DATABASE、DB_USERNAMEの箇所に.envで入力した値が設定されます。

`php artisan migrate`を実行し、データテーブルを作成します。
MySQLにログインし、テーブルが作成されていれば完了です。

<br>

### エラー発生

```
Illuminate\Database\QueryException  : SQLSTATE[42000]: Syntax error or access violation: 1071 Specified key was too long; max key length is 767 bytes (SQL: alter table `users` add unique `users_email_unique`(`email`))
```

原因はLaravel5.4から標準charasetがutf8mb4に変わったことで一文字数のByte数が4byteに増えたことです。

マイグレーションファイルのVarcharカラムは大きさを指定しなかった場合に、Varchar(255)のカラムが作成されます。

<br>

MySQL5.7以前のバージョンでは PRIMARY_KEYとUNIQUE_KEYをつけたカラムは最大767Byteしか入りません。

4Byte 255文字では767Byteを超えてしまうためこのようなエラーが発生します。

<br>

対策としてはMySQLのバージョンを最新にする、Charasetを変更する方法がありますが、今回はカラムの最大長を変更し、767Byte以上の文字列が入らないようにします。

<br>

`laravel_test/app/Providers/AppServiceProvider.php`内に以下を追記します。

```
use Illuminate\Support\Facades\Schema;

public function boot()
{
    Schema::defaultStringLength(191);
}
```
この状態でマイグレーション実行すると正常に動作が完了します。

<br>

## laravel authの作成

laravel 6.0からは`php artisan make:auth`でログイン機能の作成ができません。
    
laravel/uiとして機能が分離した
$ composer require laravel/ui "1.x" --dev

`Package manifest generated successfully.`と表示されれば完了です。
<br>

laravel/uiがインストールされた状態で、
```
php artisan ui bootstrap --auth
```
以下のように表示されれば完了です。
```
Bootstrap scaffolding installed successfully.
Please run "npm install && npm run dev" to compile your fresh scaffolding.
Authentication scaffolding generated successfully.
```

`npm install`と`npm run dev`を行うよう書かれているので実行します。


```
npm install
```

<br>

#### 以下のメッセージが表示された場合
```
found 1 high severity vulnerability
    run `npm audit fix` to fix them, or `npm audit` for details

npm audit
# Run  npm install --save-dev axios@0.21.1  to resolve 1 vulnerability
┌───────────────┬──────────────────────────────────────────────────────────────┐
│ High          │ Server-Side Request Forgery                                  │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ Package       │ axios                                                        │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ Dependency of │ axios [dev]                                                  │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ Path          │ axios                                                        │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ More info     │ https://npmjs.com/advisories/1594                            │
└───────────────┴──────────────────────────────────────────────────────────────┘
```
脆弱性が確認されたとの内容なので、要求されたコマンドを実行し解消させます。
```
npm install --save-dev axios@0.21.1
```

```
npm run dev
```
開発用にビルドします。
<br>
問題なくLaravelの認証機能の作成ができました。
<br>
以下のコマンドで一度にインストールと開発用ビルドを行うこともできます。
```
npm install && nup run dev
```

    
<br>

## ゲストOSでLaravelを動かす準備をします。
### MySQLでデータベースの作成
```
[vagrant@localhost ~]$ sudo systemctl start mysqld
```
コマンドを実行し、MySQLを起動します。
<br>
パスワードを入力し、ログインした後にcreateコマンドを使い、データベースを作成します。
```
create database laravel_test
```

`cd`コマンドでLaravel＿testのディレクトリに移動し、以下のコマンドを実行。

```
php artisan migrate
```
テーブルが作成されていれば完了です。

### Nginxの設定
```
[vagrant@localhost ~]$ sudo vi /etc/nginx/conf.d/default.conf
```
以下のように修正します。
```
server {
    listen       80;
    server_name  192.168.33.19;
    root /vagrant/laravel_test/public;
    index  index.html index.htm index.php;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        #root   /usr/share/nginx/html;
        #index  index.html index.htm;
        try_files $uri $uri/ /index.php$is_args$args;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }   

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
    #    root           html;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /$document_root/$fastcgi_script_name;
        include        fastcgi_params;
    }

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

```
[vagrant@localhost ~]$ sudo systemctl start nginx 
```
Nginxを起動します。

<br>

#### エラー発生。
```
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
nginxの起動に失敗しました。

```
[vagrant@localhost ~]$ sudo nginx -t
```
エラー内容が表示されます。

```
nginx: [emerg] unexpected "}" in /etc/nginx/conf.d/default.conf:14
nginx: configuration file /etc/nginx/nginx.conf test failed
```
14行目に原因があるようです。
```
try_files $uri $uri/ /index.php$is_args$args
```
`;`がありませんでした。修正しましょう
```
try_files $uri $uri/ /index.php$is_args$args;
```
<br>

### Nginxでphpアプリケーションを動かすにはphp-fpmという拡張モジュールの設定が必要です。
<br>
Nginxはこのphp-fpmとセットで動作するためです。
<br>
php-fpmの設定ファイルを編集します。
```
[vagrant@localhost ~]$ sudo vi /etc/php-fpm.d/www.conf
```
24行目
```
user = apache
```
以下に編集
```
user = nginx
```
26行目
```
group = apache
```
以下に編集
```
group = nginx
```


nginxとphp-fpmを再起動しましょう
```
[vagrant@localhost ~]$ $ sudo systemctl restart nginx
[vagrant@localhost ~]$ $ sudo systemctl start php-fpm
```
`403 forbidden`と表示され、画面が表示されません。
```
[vagrant@localhost ~]$ sudo firewall-cmd --add-service=http --zone=public --permanent
[vagrant@localhost ~]$ sudo firewall-cmd --reload
[vagrant@localhost ~]$ sudo systemctl restart nginx
```
```
[vagrant@localhost ~]$ getenforce
Enforcing
```
```
[vagrant@localhost ~]$ sudo setenforce Permissive
Permissive
```
こちらで設定をPermissiveに設定することでアクセスが可能になります。
<br>
何度も起動時にコマンドでPermissiveにする必要があるので、設定ファイルの内容を書き換えましょう。
```
[vagrant@localhost ~]$ sudo vi /etc/selinux/config
```
```
SELINUX=enforcing
```
以下のように修正
```
SELINUX=disabled
```
192.168.33.19へアクセス
<br>
エラーが発生してしまいました。
```
The stream or file "/vagrant/laravel_test/storage/logs/laravel.log" could not be opened in append mode: failed to open stream: Permission denied
```
```
[vagrant@localhost ~]$ sudo vi /etc/php-fpm.d/www.confで
```
php-fpmの設定ファイルのuser, groupの箇所をnginxに変更しましたが、ファイルとディレクトリの実行userとgroupにnginxが許可されていないため、エラーが起きてしまいます。

ゲストOS内のlaravel_testディレクトリ内で以下のコマンドを実行します。
```
[vagrant@localhost laravel_test]$ ls -la  | grep storage && ls -la storage/ | grep logs && ls -la storage/logs/ | grep laravel.log
drwxr-xr-x. 1 vagrant vagrant    160 Feb  6 01:50 storage
drwxr-xr-x. 1 vagrant vagrant 128 Feb  5 16:32 logs
-rw-r--r--. 1 vagrant vagrant 17385 Feb  6 11:37 laravel.log
```
`storage、log`ディレクトリ、`laravel.log`ファイルも`user`が`vagrant`となっていますが、これではユーザー権限を持って`laravel.log`ファイルへの書き込みができません。

`laravel_test`ディレクトリ内で以下のコマンドを実行し、nginxユーザーでも書き込みができるように権限を与えましょう。
```
[vagrant@localhost ~]$ sudo chmod -R 777 storage
```
```
[vagrant@localhost laravel_test]$ ls -la  | grep storage && ls -la storage/ | grep logs && ls -la storage/logs/ | grep laravel.log
drwxrwxrwx. 1 vagrant vagrant    160 Feb  6 01:50 storage
drwxrwxrwx. 1 vagrant vagrant 128 Feb  5 16:32 logs
-rwxrwxrwx. 1 vagrant vagrant 17385 Feb  6 11:37 laravel.log
```

再びhttp://192.168.33.19へアクセスします。
問題なくlaravelの画面が表示されます。

<br>

### 参考記事
[Vagrantのおけるポートフォワーディングではまった](https://qiita.com/tiruka/items/e0a03cbd71b842fef57e)

[vagrant コマンド（起動・再起動・シャットダウン等々）備忘録](https://qiita.com/htk_jp/items/929875eaa5afc71c25ed)

[# Vagrant + VirtualBOx で 最新のCentOS7 vbox(centos/7 2004.01)でマウントできない問題](https://qiita.com/mao172/items/f1af5bedd0e9536169ae)

[VagrantでCentOS7+Apache2.4+PHP7.3+Laravel5.7環境構築](https://qiita.com/okdyy75/items/d3492e4ea7136d3b6ffc)

[Laravel/ui Bootstrapの導入](https://www.techpit.jp/courses/42/curriculums/45/sections/362/parts/1144)

[Laravel5.4以上、MySQL5.7.7未満 でusersテーブルのマイグレーションを実行すると Syntax error が発生する](https://qiita.com/beer_geek/items/6e4264db142745ea666f)

