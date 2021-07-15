# ◯ログイン機能実装手順書

## 概要
--------------
VirtualBoxとVagrantを使用して環境を構築し、ログイン機能を実装する。

## バージョン一覧
--------------
|     | バージョン |
| --- | --- | 
|  MySQL | 5.7 | 
| PHP | 7.3 | 
| Laravel | 6.0 | 
| centOS7 | 7.9.2009 |
| Nginx | 1.21.1 |

## 条件
--------------
- Vagrant を使用すること
- ipは 192.168.33.19 とすること
- OSは CentOS7
- Webサーバは Nginx

## 作業手順
--------------


### ＜環境構築＞
### VirtualBoxのインストール

[Virtual Box公式](https://www.vagrantup.com/)
※ Vagrantの最新バージョンがVirtualBoxの最新バージョンに対応していないため、Virtual Boxはver6.0.14をインストールするようにしてください。

上記のURLでは**OS X hosts**を選択しましょう。 

以下のコマンドを実行してVirtualBoxのウィンドウが表示されれば正常にインストールされています。

```
$ virtualbox
```

※ コマンド実行後は入力を受け付けない状態となるため、 Control + c を押してください。


### vagrantのインストール

```
$ brew cask install vagrant
```

homebrewのバージョンが新しいと、caskコマンドがオプションになっており、上記のコマンドでエラーになる場合があります。その時は下記のコマンドでインストールしましょう。

```
$ brew install --cask vagrant
```

下記でインストールできているか確認してください。

```
$ vagrant -v
```

### vagrant boxのダウンロード

```
vagrant box add centos/7
```

コマンドを実行すると、下記のような選択肢が表示されます。

```
1) hyperv
2) libvirt
3) virtualbox
4) vmware_desktop

Enter your choice: 3
```

今回使用するソフトはVirtualBoxのため、3を選択してenterを押しましょう。
下記のように表示されたら完了です。

```
Successfully added box 'centos/7' (v1902.01) for 'virtualbox'!
```

### Vagrantの作業ディレクトリを用意する

```
mkdir login
```

作成したフォルダの中で以下のコマンドを実行します。

```
cd login
# vagrant init box名 先ほどダウンロードしたboxを使用することになります
vagrant init centos/7

# 実行後問題なければ以下のような文言が表示されます
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
```

### Vagrantfileの編集

```
# 変更点①
config.vm.network "forwarded_port", guest: 80, host: 8080

# 変更点②
config.vm.network "private_network", ip: "192.168.33.19"

# 変更点③
config.vm.synced_folder "../data", "/vagrant_data"
# ↓ 以下に編集
config.vm.synced_folder "./", "/vagrant", type:"virtualbox", :mount_options => ["dmode=777,fmode=777"]
```

変更点全てをコメントインしてください。

### Vagrant プラグインのインストール

```
vagrant plugin install vagrant-vbguest

vagrant plugin install sahara
```

### Vagrantを使用してゲストOSの起動,ゲストOSへのログイン

```
vagrant up

vagrant ssh
```

※ vagrant upで共有フォルダのマウントができなかった場合の対処方法は下記URLを参考にしてください。

[対処方法](https://qiita.com/mao172/items/f1af5bedd0e9536169ae)

### パッケージをインストール

```
sudo yum -y groupinstall "development tools"
```

### PHPのインストール

```
sudo yum -y install epel-release wget
sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
sudo rpm -Uvh remi-release-7.rpm
sudo yum -y install --enablerepo=remi-php73 php php-pdo php-mysqlnd php-mbstring php-xml php-fpm php-common php-devel php-mysql unzip
php -v
```

### composerのインストール

```
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"

# どのディレクトリにいてもcomposerコマンドを使用できるようfileの移動を行います
sudo mv composer.phar /usr/local/bin/composer
composer -v
```

### laravelインストール

loginディレクトリに移動してください。

```
cd login
```

laravelの6系をインストールしてください。

```
composer create-project --prefer-dist laravel/laravel laravel_app "6.*"
```



### Nginxのインストール

ファイル作成

```
$ sudo vi /etc/yum.repos.d/nginx.repo
```

書き込む内容は以下になります。

```
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/
gpgcheck=0
enabled=1
```

書き終えたら保存して、以下のコマンドを実行しNginxのインストールを実行します。

```
$ sudo yum install -y nginx
$ nginx -v
```

Nginxのバージョンは確認できたでしょうか？
ではNginxの起動をしましょう。

```
$ sudo systemctl start nginx
```

ブラウザにて http://192.168.33.19 と入力し、NginxのWelcomeページが表示されたら成功です。

### Laravelを動かす

Nginxの設定ファイルを編集していきます。

```
$ sudo vi /etc/nginx/conf.d/default.conf
```

```
server {
  listen       80;
  server_name  192.168.33.19; # Vagranfileでコメントを外した箇所のipアドレスを記述してください。
  # ApacheのDocumentRootにあたります
  root /vagrant/laravel_app/public; # 追記
  index  index.html index.htm index.php; # 追記

  #charset koi8-r;
  #access_log  /var/log/nginx/host.access.log  main;

  location / {
      #root   /usr/share/nginx/html; # コメントアウト
      #index  index.html index.htm;  # コメントアウト
      try_files $uri $uri/ /index.php$is_args$args;  # 追記
  }

  # 省略

  # 該当箇所のコメントを解除し、必要な箇所には変更を加える
  # 下記は root を除いたlocation { } までのコメントが解除されていることを確認してください。

  location ~ \.php$ {
  #    root           html;
      fastcgi_pass   127.0.0.1:9000;
      fastcgi_index  index.php;
      fastcgi_param  SCRIPT_FILENAME  /$document_root/$fastcgi_script_name;  # $fastcgi_script_name以前を /$document_root/に変更
      include        fastcgi_params;
  }

  # 省略
```

Nginxの設定ファイルの変更は、以上です。
次に php-fpm の設定ファイルを編集していきます。

```
$ sudo vi /etc/php-fpm.d/www.conf
```

変更箇所は以下になります。

```
;24行目近辺
user = apache
# ↓ 以下に編集
user = nginx

group = apache
# ↓ 以下に編集
group = nginx
```

以下のコマンドを実行して nginx というユーザーでもログファイルへの書き込みができる権限を付与してあげましょう。

```
$ cd /vagrant/laravel_app
$ sudo chmod -R 777 storage
```

設定ファイルの変更に関しては、以上となります。
では早速起動しましょう(Nginxは再起動になります)。

```
$ sudo systemctl restart nginx
$ sudo systemctl start php-fpm
```

http://192.168.33.19 を入力して確認してください。laravelのwelcomeページが表示されていれば大丈夫です。

### データベースのインストール

今回インストールするデータベースはMySQLとなります。versionは5.7を使用します。

```
sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm
sudo yum install -y mysql-community-server
mysql --version
```

versionの確認ができましたらインストール完了です。
次にMySQLを起動し接続を行います。

```
sudo systemctl start mysqld
```

パスワードの確認

```
sudo cat /var/log/mysqld.log | grep 'temporary password'
```

※ 出力結果の**root@localhost:**のあとがパスワードになります。

では先程出力したランダム文字列をコピー後、再度以下のコマンドを実行し、パスワード入力時にペーストしてください。

```
$ mysql -u root -p
$ Enter password:
mysql >
```

問題なく接続できたでしょうか？
次に接続した状態でpasswordの変更を行います。

```
mysql > set password = "新たなpassword";
```

### データベースの作成

```
mysql > create database laravel_app;
```

### login機能の実装

laravel_appディレクトリ下の .env ファイルの内容を以下に変更してください。

```
DB_PASSWORD=
# ↓ 以下に編集
DB_PASSWORD=登録したパスワード
```

Laravelはインストールしてあるのでマイグレーションまで実行しておきます。

```
$ php artisan migrate
```

laravel/uiライブラリをインストールします。

```
composer require laravel/ui "^1.0" --dev

php artisan ui vue --auth
```

以上で作業は終了です。

## 環境構築の所感
--------------
・手順書を先に書こうとしたが、先に実行してからじゃないと正解が分からないため、実行しながら手順書を書く必要があることがわかった。
・エラーが多数出る、また複雑なため、作業に時間がかかった。
・それぞれのローカル環境によって違うエラーが出ていたので、手順書で誰もが同じ環境を構築できる様にすることが重要だと理解した。
・ゲストOS側で共有フォルダのパーミッションは変更できない。＞＞＞ホストOSで変更できる。

## 参考サイト
--------------
[CentOSでremiとEPELを使いphpのバージョンをアップ/ダウングレードする方法](https://www.geekfeed.co.jp/geekblog/centos-remi-epel-php)
[Laravel6 ログイン機能を実装する](https://qiita.com/ucan-lab/items/bd0d6f6449602072cb87)
[Giztech](https://giztech.gizumo-inc.work/lesson/18/217)