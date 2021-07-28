# メモ
## 前提
- MacOS
- Docker
- インターネット接続可能（Google Container Registryへのアクセス）
- Spring Boot
- Spring Data Cloud Spanner

## 準備
### Cloud CLIのインストール（Dockerイメージ）
- https://cloud.google.com/sdk/docs/downloads-docker
  - 「指定した Docker イメージのインストール」でpullしてくるだけで良い。
  
### Spannerのエミュレータのインストール（Dockerイメージ）
- https://cloud.google.com/spanner/docs/emulator#docker
- SpannerのDockerは常に使うのでずっと動かしておく。
- 途中「エミュレータで gcloud CLI を使用する」でCloud CLIが必要になるため、
  - Cloud CLIをインストールする（前出）。
  - Cloud CLIをDockerに入れているので、以下に読み替えて、Docker内でエミュレータ切り替え操作を実施。
 
     ```
     docker run --rm -it gcr.io/google.com/cloudsdktool/cloud-sdk:latest
     gcloud config configurations create emulator
     gcloud config set auth/disable_credentials true
     gcloud config set project demo-spanner
     gcloud config set api_endpoint_overrides/spanner http://host.docker.internal:9020/
     ```

    - `host.docker.internal`に変更している理由について。CLIのDockerから見たSpannerエミュレータの宛先はlocalhostではないため。
      - https://qiita.com/ijufumi/items/badde64d530e6bade382#%E3%81%9D%E3%81%AE5hostdockerinternal-%E3%82%92%E4%BD%BF%E3%81%86-20190916%E8%BF%BD%E8%A8%9820200227%E7%B7%A8%E9%9B%86
  - CLI Docker内のまま、インスタンスを作成。

     ```
     gcloud spanner instances create test-instance --config=emulator-config --description="Test Instance" --nodes=1
     ```
  - CLI Docker内のまま、databaseを作成。

     ```
     gcloud spanner databases create test-database --instance=test-instance
     ```
   	 - SpannerにはRDBのスキーマ相当の概念が無い模様
   	     - スキーマという単語 は出てくるが、別の意味ぽい。 

### TBLを準備する
- DDLやDMLはgcloud spannerで行う（サードパーティのspanner-cliもあるが）
  - https://cloud.google.com/spanner/docs/modify-gcloud
  - https://cloud.google.com/sdk/gcloud/reference/spanner
  - https://cloud.google.com/sdk/gcloud/reference/spanner/databases/ddl/update#DATABASE
- todoテーブルの削除と作成。CLIのDocker内で実行。

  ```
  gcloud spanner databases ddl update test-database --ddl="DROP TABLE todo"  --instance=test-instance
  gcloud spanner databases ddl update test-database --ddl="CREATE TABLE todo(id STRING(36), title STRING(30), finished BOOL, created_at TIMESTAMP) PRIMARY KEY (id)"  --instance=test-instance 
  ```

## サンプルAPの作成や実行
###  Spring Initializrでベース作成
- Spring Cloud GCPはSpring Boot 2.5.0に未対応なので、Spring Boot 2.4.xにする必要がある。
- Spring Cloud GCPを選択してZipをDLする。
  - pomのgroupIdは`com.google.cloud`だが、これは正しい（リファレンスだと`org.springframework.cloud`となっているが、リファレンス側の最新化漏れぽい）。
- Spring Cloud GCP全体のstarterが設定されているので、`spring-cloud-gcp-starter-data-spanner`に修正。

### Spannerエミュレータ接続設定
- application.propertiesには、エミュレータ準備時に設定した以下の値を入れる。

  ```
  spring.cloud.gcp.spanner.instance-id=test-instance
  spring.cloud.gcp.spanner.database=test-database
  spring.cloud.gcp.spanner.project-id=demo-spanner
  spring.cloud.gcp.spanner.emulator.enabled=true
  ```

### サンプルAPのRepositoryやロジック実装
- TERASOLUNA FrameworkのTODOアプリのServiceを真似て作成。
  - https://terasolunaorg.github.io/guideline/5.7.0.RELEASE/ja/Tutorial/TutorialTodo.html#service
- First Stepなので、JOINが無い、1TBLのAPとした。
- 簡単のため、Webアプリではなく、コマンドラインAP（ApplicationRunner）で実装。

### サンプルAPの実行
- DemoApplicationをSpringBootで実行（STSからでOK）
  - `ApplicationRunner`で起動時にレコード追加や削除を適当にしている。結果がログに出るので、正常に動いたかを確認。
  - Spring Cloud SpannerのログをDEBUGレベルにしているので、邪魔な場合はログレベルを修正すること（application.properties）。
  -  最後にレコードをすべて削除するサンプルAPにしているが、エラーなどで途中でアベンドしDBにデータが残ってしまった場合、次回実行時に想定通り動かない。「TBL準備」のテーブル削除＆作成手順によりレコードをリセットすること。
     - RDB向けの`schema.sql`や`data.sql`相当機能が無いのが辛い。 
  
## 公式サンプルを試す（TODO）
- https://github.com/GoogleCloudPlatform/spring-cloud-gcp/tree/main/spring-cloud-gcp-samples/spring-cloud-gcp-data-spanner-sample
- git cloneしてきて、`spring-cloud-gcp-data-spanner-sample`を`mvn install`
  - mvnでartifactがないとエラーが出てくる。SNAPSHOTが格納されているrepositoryを追加必要ぽく面倒なので、一旦保留。
