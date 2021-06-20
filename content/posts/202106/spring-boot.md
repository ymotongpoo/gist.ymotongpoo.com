---
title: "Spring Bootを試した"
date: 2021-06-18T11:02:34+09:00
draft: false
tags: ["spring-boot", "spring", "java", "docker", "programming"]
---

## SpringとSpring Boot

この辺の記事を読んだ。

* https://licensecounter.jp/devops-hub/blog/spring-boot1/
* https://licensecounter.jp/devops-hub/blog/spring-boot2/
* https://qiita.com/suke_masa/items/9dd3300c3190d6445ff8

ざっくりとした理解は

* Spring
  * 様々なコンポーネント群からなるフレームワーク
  * 他のJavaフレームワークと違い、ミドルウェア含めたFat JARを作成し、手軽に起動できるようにしている。
  * 独立したJARを作するので、他のアプリケーションと共有のライブラリなどがなく、依存関係で揉めることがない。
* Spring Boot
  * プロジェクトを作成する際にSpringの中から必要なコンポーネントを選択して最小セットのskaffoldを作成してくれる
  * 後述のSping InitializrやIDE組み込みのツールで使える

## Spring Initializr

* https://start.spring.io/

IntelliJ UltimateだったらテンプレートでSpring Bootの設定ができるらしいけど、IntelliJ CEだとプラグインは野良のものしか無いようなので、Spring Initializrを使って行う。

* https://spring.pleiades.io/quickstart

ここを読んだらGETを受け付けるREST APIのHello Worldはすぐにできた。

```console
./mvnw spring-boot:run
```

## コンテナイメージの作成

とりあえず何も考えずにDockerをつかてビルドしたんだけど、ベースイメージの選択ではまった。

```dockerfile
FROM maven:3.8-openjdk-11 as builder
WORKDIR /build
COPY . .
RUN mvn install

FROM gcr.io/distroless/java-debian10
COPY --from=builder /build/target/demo-0.1.0.jar /jar/demo-0.1.0.jar
EXPOSE 8080
CMD ["/jar/demo-0.1.0.jar"]
```

Distrolessの場合は `java -jar` が指定された状態なので `CMD` に渡すのはJARファイルのパス