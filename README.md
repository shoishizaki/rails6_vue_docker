# Rails6+Vue+MySQL を Docker で環境構築する

## ゴール

Rails をサーバーサイド、Vue をフロントエンドとして使い Docker で環境構築を行う  
　データベースは MySQl を用いる

## 手順

1.Dockerfile  
 FROM ruby:2.7  
 RUN apt-get update -qq && apt-get install -y nodejs yarnpkg  
 RUN ln -s /usr/bin/yarnpkg /usr/bin/yarn  
 RUN mkdir /app  
 WORKDIR /app  
 COPY Gemfile /app/Gemfile  
 COPY Gemfile.lock /app/Gemfile.lock  
 RUN bundle install  
 COPY . /app

Add a script to be executed every time the container starts.← コメント  
COPY entrypoint.sh /usr/bin/  
RUN chmod +x /usr/bin/entrypoint.sh  
ENTRYPOINT ["entrypoint.sh"]  
EXPOSE 3000

Start the main process.← コメント  
CMD ["rails", "server", "-b", "0.0.0.0"]

2.初期 Gemfile  
source 'https://rubygems.org'  
gem 'rails', '~>6'

3.Gemfile.lock を作成（空で良い）

4.entrypoint.sh ←Dockerfile で ENTRYPOINT として定義している entrypoint.sh  
!/bin/bash← コメント  
set -e

Remove a potentially pre-existing server.pid for Rails.← コメント  
rm -f /app/tmp/pids/server.pid

Then exec the container's main process (what's set as CMD in the Dockerfile).← コメント  
exec "\$@"

5.docker-compose.yml  
version: '3'  
services:  
 db:  
 image: mysql:8.0  
 volumes:

- ./tmp/db:/var/lib/mysql  
  environment:
- MYSQL_ALLOW_EMPTY_PASSWORD=1  
  web:  
  build: .  
  command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"  
  volumes:
- .:/app  
  ports:
- "3000:3000"  
  depends_on:
- db

  6.rails new のコマンドを web コンテナ上で実行して Rails のファイル群を生成する。  
  \$ docker-compose run web bundle exec rails new . --force --database=mysql

  7.Rails のファイル群が rails new コマンドによって出来上がったので build 　する。  
  \$ docker-compose build

  8.DB ホスト名変更  
  config/database.yml の host の部分を db に置き換える。

  9.build 後に docker-compose up する。  
  \$ docker-compose up

  10.DB コンテナで mysql クライアント起動  
  \$ docker-compose exec db bash

  11.db コンテナの bash を起動後に mysql コマンドで接続する。  
  mysql -u root

  12.認証方式変更 SQL  
  mysql> select User,Host,plugin from mysql.user;

全て caching_sha2_password に設定されています。これを mysql_native_password に変更する。

mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '';

13.DB が作成されていないよ、というメッセージが出るので db:prepare でテーブルを作成する。  
\$ docker-compose exec web bundle exec rails db:prepare

これで Rails のホーム画面が表示されるようになる。

14.Vue.js の導入  
\$ docker-compose run web rails webpacker:install:vue

15.Vue.js ファイルのコンパイル  
\$ docker-compose run web bin/webpack

## 参考文献

https://blog.toshimaru.net/rails-on-docker-compose/

https://qiita.com/nagamine09/items/5bb95ef4c3714ac0a483

https://www.koatech.info/blog/rails-docker-webpacker/
