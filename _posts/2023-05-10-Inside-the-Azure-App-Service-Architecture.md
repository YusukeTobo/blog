---
title: "[日本語訳] Inside the Azure App Service Architecture"
author_name: "Kohei Mayama"
tags:
    - App Service
    - Function App
    - Web Apps
---

## はじめに

本記事は、MSDN マガジン の2017 年 2 月に投稿されました Azure App Service の基本的なアーキテクチャを公開している [Inside the Azure App Service Architecture](https://learn.microsoft.com/ja-jp/archive/msdn-magazine/2017/february/azure-inside-the-azure-app-service-architecture) の日本語抄訳としてご案内いたします。

Azure App Service の基本的なアーキテクチャの解説をしておりますが、2017 年の記事であるため内容が変更されている場合がございます。


## Azure App Service アーキテクチャの内部

Azure App Service は優れた Platform as a Service (PaaS) で開発者が Web、モバイルそして API アプリケーションを構築するためにアプリケーションプラットフォームを提供しております。シンプルなマーケティングやデジタル展開するアプリケーションから、拡張性がある e-コマースソリューションやハイパースケールでカスタマイズ可能なアプリケーションまで及びます。

App Service はフルマネージドです。つまり、アプリケーションを実行するためのインフラ（サーバ）を管理する業務はございません。OS プラットフォームやフレームワークのパッチをサーバに適用しメンテナンスをする必要の心配はございません。アプリケーションは仮想サーバ上にて動作しますが、アプリケーションを動かすためのサーバインスタンスの最大値を設定するのみ注意する必要があります。プラットフォームでは、アプリケーションにて複数のサーバインスタンスを必要とする場合にスケーリングをすると同時に、ロードバランサがトラフィックを複数のインスタンスへ振り分けます。

App Service チームは実装の詳細を隠すようにしておりますが、内部でどのように機能するか知ることが良いでしょう。この記事では、Web Apps の基本的な内部アーキテクチャ（どのようなサービスが起動し動作するか）や特定のシナリオにおけるベストプラクティスを提供します。


## グローバルと地理的分散のアーキテクチャ

クラウドコンピューティングは早くスケールを行い、限りないキャパシティを持っています。クラウドのスケールは、PC 画面の動きと同様の説明をすることができます。PC の画面を遠くから見ると、鮮明になめらかな画像を見ることができます。近くにて見ると、多くの小さなピクセルとして構成されています。クラウドも画像と同様で、多くのサーバによって構成されています。App Service は、サーバの束を "スケールユニット" と呼ばれる １ つのクラスタとします。Azure データセンター内には、世界中にそのようなスケールユニットが多くございます。

Azure の一部として、App Service は世界中に拠点があります。Azure がサポートする全てのリージョンには、お客様のワークロード（アプリケーション）を実行する App Service スケールユニットとリージョナルコントロールユニットのセットがあります。コントロールユニットはお客様に対して（誤動作が発生するまで）わかりやすく、プラットフォームの一部として動作します。全ての API 呼び出しゲートウェイとして使用されている特別なコントロールユニットがあります。例えば、お客様が Azure Portal を介したり、コマンドラインインターフェイスもしくは直接 Azure REST API を使用して新しいアプリケーションを作成をリクエストした際に、そのリクエストは主要の Azure エンドポイント (management.azure.com) へルーティングされます。Azure Resource Manager, もしくは ARM ( https://bit.ly/2i6UD07 ) はお客様のアプリケーション内の様々な Azure リソースを単一のグループとして操作できます。ARM によって定義された API を利用すると Azure リソースを管理できます。ARM は実際に個別のリソースを管理できません。Azure の各サービスは独自の API 実装をしておりますが、ARM により API 呼び出しを委任しております。App Service の場合、ARM は App Service API の呼び出しを App Service Geo-マスター へ転送します。

Geo-マスター は世界中全てのスケールユニットという意味があります。例えば、新しい App Service アプリケーション（もしくは Web サイト）を作成する時に、Geo-マスター は最も適したスケールユニットを探し、その後、作成要求に適したスケールユニットへ転送します。スケールユニットは、新しいアプリケーションのプロビジョニングとアプリケーションに必要なリソースの割り当てを実施します、図 1 は新しいアプリケーションを作成中について示しております。

![image-991e4c0f-a33a-4ad4-acb7-1858cfadeb57.png]({{site.baseurl}}/media/2023/05/image-991e4c0f-a33a-4ad4-acb7-1858cfadeb57.png)

図 1: App Service スケールユニットのグローバルディストリビューション

新しいアプリケーションを作成する処理は以下の通りです。

1. ユーザが新しいサイトを作成するためのリクエストをします
2. ARM はユーザの操作を許可（今回は作成）するためにリソースへアクセスできることを確認し、リクエストを Geo-マスター へ転送します
3. Geo-マスター は、ユーザから転送されたリクエストにて一番適したスケールユニットを探します
4. スケールユニットは新しいアプリケーションを作成します
5. Geo-マスター はアプリケーション作成リクエストが成功したことを通知します

App Service は多くのスケールユニットがあることを理解することが重要ですが、アプリケーションは単一の App Service スケールユニット内で実行されます。Azure Traffic Manager を使用して複数のリージョンで実行したとしても、アプリケーションは ２ つ以上のスケールユニットで実行されます。しかしながら、個々のスケールユニットの観点からは、アプリケーションは 1 つのスケールユニットに制限されます。


## App Service スケールユニットとは？

App Service スケールユニットとはアプリケーションをホストおよび実行するサーバをまとめたものです。一般的なスケールユニットは、1000 以上のサーバを持っています。サーバをクラスタリングしたことにより、スケールの効率的使用やインフラストラクチャの再利用が可能となります。App Service スケールユニットの基本的な構成要素は、Azure Cloud Service デプロイメントです。（App Service は、2012 年 6 月にプレビュー版としてリリースされました。）どのスケールユニットも自律的であり、単独で動作します。

## スケールユニットの主要な構成要素

スケールユニットの主な機能はお客様のアプリケーションをホストし実行することです。アプリケーションは Windows サーバ上で実行され、Web Worker、Worker と呼ばれています。スケールユニット内のサーバの大部分は、Worker となります。しかしながら、スケールの単位では、App Service によって提供される機能を実現するために必要な追加のサポートサーバが含まれています。サポートサーバは Role を持っており、各 Role は冗長性と拡張性のために複数のインスタンス上にデプロイされます。

## FrontEnd

FrontEnd は、レイヤー (OSI レイヤー 7) 、プロキシとしての振る舞いをし、異なるアプリケーションとそれぞれの Worker 間で HTTP リクエストの呼び出しを分散させます。現在、App Service ロードバランサのアルゴリズムは、特定のアプリケーションに割り当てたサーバ間にて、単純なラウンドロビンとしております。


## Web Workers

複数の Worker は、App Service スケールユニットのバックボーンでございます。お客様自身のアプリケーションを実行することができます。

App Service を使用することで、アプリケーションの実行方法を選択することができます。共有サーバまたは専用サーバ上にてアプリケーションを実行を選択することができます。これを行うには、App Service Plan にて選択します。App Service Plan は一連の性能、機能やサーバへの割り当てを定義することができます。共有 Worker は複数の異なるお客様にてアプリケーションをホストをし、専用 Worker は 1 人のお客様にて、1つ 以上のアプリケーションを実行することが保証されております。複数の専用サーバの種類やサイズがあり、お客様より選択可能です。大きなサーバのサイズは、アプリケーションに割り当てるために、より多くの CPU や メモリのリソースが利用可能です。App Service Plan は、アプリケーションに対して事前に割り当てられたサーバの数を定義します。

App Service スケールユニットは、事前にプロビジョニングされた Workers のプールがいくつかあり、お客様のアプリケーションをホストする準備をしているのを 図 2 のセクション 1 に示しております。専用の App Service Plan で 2 台のサーバサイズを定義をすると、図 2 のセクション 2 に示しますように App Service は 2 台のサーバを割り当てます。次に、お客様の App Service Plan にてスケールアウトをすると（例えば、さらに 2 つの Worker を追加をする）、図 2 のセクション 3 に示すように、利用可能な Worker は即座に使用できる Worker のプールから割り当てられます。Worker は事前にプロビジョニングしウォームスタンバイであるので、アプリケーションを Worker へデプロイするだけとなっております。アプリケーションがデプロイされると、Worker はローテーションに入り、FrontEnd がアプリにトラフィックを割り当てます。これらのプロセス全体は通常、数秒かかります。

![image-67787ae8-caf4-4e09-8037-7a74ea0f1ac1.png]({{site.baseurl}}/media/2023/05/image-67787ae8-caf4-4e09-8037-7a74ea0f1ac1.png)

図 2: App Service スケールユニット内のサーバアプリケーション処理

図 2 のセクション 4 では、複数の色や四角で囲まれている App Service Plan を示しております。複数のお客様によって Worker のプールを共有している App Service Plan を表しております。


## File Servers

複数のアプリケーションは、HTML、js ファイル、画像、コードファイルなどのコンテンツやアプリケーションが機能するためのコンテンツを保持するためにストレージが必要となります。ファイルサーバは Azure Storage blob をマウントし、ネットワークドライブとして Worker へ公開します。Worker はそのネットワークドライブをローカルとしてマッピングし、アプリケーションがローカルディスクを使用するサーバで実行されている場合と同様にして、Worker で実行されているアプリケーションは "ローカル" ドライブとして使用できます。アプリケーションによって実行されるファイルに関する読み取り/書き込みはすべてファイルサーバを通しています。

## API Controllers

API Controllers は、App Service Geo-マスター の拡張機能としております。Geo-マスター は、すべての スケールユニットにわたる App Service アプリケーションを認識しており、アプリケーションに影響を与える管理操作を実際に実行しているのが、API コントローラーになります。Geo-マスター は、API Controllers を返して API の実行をスケールユニットに委任します。例えば、Geo-マスター が API を呼び出して新しいアプリケーションを作成すると、API Controller がスケールユニットにてアプリケーションを作成するために必要な手順を行います。アプリケーションのリセットを Azure Portal を使用したときに、アプリケーションに現在割り当てられている Web Worker を再起動するように通知を行うのが、API Controller です。

## Publisher

Azure App Service は、アプリケーションへの FTP アクセスをサポートしています。アプリケーションのコンテンツは Azure Storage blobs と ファイルサーバによってマッピングされているため、App Service は、FTP 機能を公開する Publisher ロールを持っています。Publisher ロールにより、お客様は FTP を使用しアプリケーションコンテンツやログへアクセスすることができます。

アプリケーションのデプロイは FTP 以外にも存在しているため注意する必要がございます。一般的なデプロイ方法は Web Deploy （Visual Studio から）もしくはVisual Studio Release Manager や GitHub のような継続的デプロイがサポートされています。


## SQL Azure

各 App Service スケールユニットは、アプリケーションのメタデータを永続化するために Azure SQL Database を使用します。スケール単位に割り当てられている各アプリケーションには、SQL データベース内にて表現されています。SQL データベースはに関するランタイム情報を保持するためにも使用されます。


## Data Role

すべての role が機能するには、データベースにあるデータが必要です。例として：Web Worker は、サイトの構成情報を必要とします。FrontEnd は、HTTPリクエストを適切なサーバへ転送するために、特定のアプリケーション実行している割り当てられたサーバを知る必要があります。Controller は、お客様が実行した API 呼び出しに基づいて、データベースからデータを読み取り、更新を行います。Data Role は、SQL データベースとスケールユニット内にある他の Role 間のキャッシュレイヤーとしております。他の Role からデータ層（SQL データベース）を抽象化し、スケールとパフォーマンスをを控除させ、ソフトウェア開発やメンテナンスを簡素化します。


## あまり知られていないベストプラクティス

Azure App Service の構築方法を知ったところで、App Service チームからのヒントとコツを確認していきます。これらは、App Service エンジニアリングチームが多くのお客様のエンゲージメントから学んだ実践的な教訓です。

## 密度のコントロール

多くのお客様は 1 つの App Service Plan にて少数（10 未満）のアプリケーションを実行しています。しかしながら、お客様はより多くのアプリケーションを実行するシナリオが多く存在します。基盤となるサーバの容量を誤って飽和させないことが重要となります。

アプリケーションとコンピューティングリソースの基本的な階層から始めていきましょう。App Service Plan に関連付けられている 2 つの Web アプリケーションと 1 つのモバイルバックエンドがあります。

デフォルトでは、App Service Plan に含まれるすべてのアプリケーションは、その Service Plan に割り当てられた利用可能なコンピューティングリソース（サーバ）で実行されます。3 つのアプリケーションはすべて、二つのサーバで実行されます。App Service Plan に単一のサーバがあるケースでは、理解することは簡単です。App Service Plan のすべてのアプリケーションは単一のサーバで実行します。

App Service Plan に複数のコンピューティングリソースが割り当てられている場合に何が発生するかについては、やや直感的ではありません。例として、単一の App Service Plan に 10 個のコンピュートリソースがある場合に、その後すべてのアプリケーションが全てのコンピュートリソースで実行されます。もし、50 個のアプリケーションが App Service Plan にある場合に、50 個の全てのアプリケーションが最初のサーバで実行され、同じ 50 個が 2番目のサーバで実行され、以下同様にして 10 番目のサーバで実行されます。 10 番目のサーバも 50 個全てのアプリケーションを実行します。

一部のシナリオでは、アプリケーションが大量のコンピューティングリソースを必要とする場合、通常は HTTP リクエストの増加を処理するために、利用可能な全てのサーバ上でアプリケーションを実行する必要があります。しかしながら、App Service Plan が 1 つのサーバから多くのサーバへスケールアウトされた時に意図しない結果である場合があります。多くのアプリケーションの実行によって、App Service Plan が CPU やメモリの負荷にさらされている場合は、その App Service Plan にてサーバの数を増加しても問題は解決されません。

代わりに、各アプリケーションのトラフィック分散を考慮し、容量が少ないアプリケーションのロングテールとなっているものを個別の App Service Plan へ分割をします。個別のを App Service Plan にて、容量の大きいアプリケーションを実行することを考慮ください。前回の 50 個のアプリケーションの例を使用しますと、トラフィックパターンを分析した後、App Service Plan とコンピューティングリソースに次のように割り当てることができます。

* 40 個の容量が少ないアプリケーションを、1 つのコンピューティングリソース上の単一の App Service Planに残します。
* 5 個の中〜低容量のアプリケーションを、1 つのコンピューティングリソース上で実行される 2 番目の App Service Plan を使用します。
* 残りの 5 個のアプリケーションは、容量を多く使用していることが分かりました。各アプリケーションは、個別の App Service Plan へデプロイします。オートスケールルールは、CPU とメモリの使用率に基づいてスケールイン/アウトするルールを設定し、最低でも 1 つのコンピューティングリソースを使用する App Service Plan を設定します。

この手法の最終的な結果は、50 個のアプリケーションは最低でも 7つのコンピューティングリソースを使用し、5 個の容量の大きいアプリケーションは負荷に基づいてオンデマンドで個別にスケールアウトするために必要な App Service Plan があります。


## アプリケーションごとのスケーリング

より効率的な方法で多くのアプリケーションを実行するためのもう 1 つの方法は、Azure App Service のアプリケーションごとにスケーリングする機能を使用することです。こちらのドキュメントでは https://bit.ly/2iQUm1S 、アプリケーションごとのスケーリングについて詳しく説明しております。（原文がリンク切れですのでそのまま掲載しておりますのでご了承ください。）アプリケーションごとのスケーリングにより、アプリケーションに割り当てられるサーバの最大数を制御でき、アプリケーションごとに制御できます。この場合、アプリケーションは定義された最大数のサーバで実行され、利用可能なすべてのサーバで実行されることではございません。

前回の 50 個のアプリケーションの例を使用すると、App Service Plan でアプリケーションごとのスケーリングが有効になっているため、50 個のアプリケーションは全て同じ App Service Plan に割り当てることができます。次に、個別のアプリケーションのスケーリング特性を変更することができます：

* 40 個の容量が少ないアプリケーションをそれぞれ最大 1 台のサーバで実行するように設定します。
* 5 個の中〜低容量のアプリケーションを、それぞれ最大 2 台のサーバで実行するように設定します。
* 残りの 5 つの容量が大きいアプリケーションは、それぞれ最大 10 台のサーバで実行するように設定します。

基盤となる App Service Plan は、最低 5 台のサーバから開始をし、次に、オートスケールルールを設定して、メモリの負荷と CPU に基づいて必要に応じてスケールアウトできます。

Azure App Service は、コンピューティングリソースへのアプリケーションの割り当てを自動的に処理します。このサービスは、個別のアプリケーションの Worker 設定の数に基づいて、実行中のアプリケーションインスタンスの最大値にて自動的に処理します。その結果、App Service Plan の Worker 数を増やしても、利用可能な新しい仮想マシンごとに 50 個のアプリケーションインスタンスを起動することはございません。

要約すると、アプリケーションごとのスケーリングは、App Service Plan に関連付けられた基盤となるサーバ上にアプリケーションを "パック" します。前述のように、 App Service Plan に関連付けられた全てのコンピューティングリソースで全てのアプリケーションが実行されるわけではございません。

## アプリケーションスロット

App Service は、"デプロイメントスロット"（ https://bit.ly/3qqHh1Y ） と呼ばれる機能がございます。簡単に説明すると、デプロイメントスロットを使用すると、本番アプリケーション以外の別のアプリケーション（スロット）が利用できます。それは、本番環境へスワップする前に新しいコードをテストするために使用できるもう 1 つのアプリケーションです。

アプリケーションスロットは、App Service で最も使用されている機能の 1 つです。しかしながら、各アプリケーションスロットは、それ自体がアプリケーションであることを理解することが重要です。つまり、アプリケーションスロットには、カスタムドメイン、異なる SSL 証明書、異なるアプリケーション設定を関連付けることができます。また、App Service Plan へのアプリケーションスロットの割り当ては、本番スロットに関連付けられた App Service Plan とは別で管理できます。

デフォルトでは、各アプリケーションスロットは本番スロットとして同じ App Service Plan 内に作成されます。容量が少ないアプリケーション、および CPU/メモリ使用率が低いアプリケーションの場合は、この手法は問題ございません。

しかしながら、App Service Plan 内の全てのアプリケーションは同じサーバで実行されるため、アプリケーションのすべてのスロットは本番環境とまったく同じサーバで実行されていることを意味しています。これにより、本番アプリケーションスロットと同じサーバで実行される非本番スロットに対して負荷テストを実行する場合、CPU やメモリ制約などの問題が発生する可能性がございます。


リソースの競合が負荷テスト実行などのシナリオのみに限定されている場合、スロットを独自のサーバセットを備えた App Service Plan に一時的に移動すると次のようになります：

* 非本番スロット用に追加の App Service Plan を作成します。注意：各 App Service Plan は、本番スロットの App Service Plan と同じリソースグループおよび同じリージョンである必要がございます。
* 非本番スロットを異なる App Service Plan へ移動し、コンピューティングを別のプールへ移動します。
* 別の App Service Plan で実行しながら、リソースを大量に消費する（またはリスクの高い）タスクを実行します。例えば、リソースの競合が発生しないため、本番スロットに悪影響を与えることなく、非本番スロットに対して負荷テストを実行できます。
* 非本番スロットを本番環境にスワップする準備ができましたら、本番環境スロットを実行している同じ App Service Plan へ戻します。その後、スロットスワップ操作を実行します。


## ダウンタイムなしで商用環境へデプロイする

App Service Plan 上で正常に実行されているアプリケーションがあり、アプリケーションを毎日更新する優れたチームがあります。この場合、少しも本番環境に直接デプロイを行いたくありません。デプロイメントをコントロールし、ダウンタイムを最小限に抑える必要があります。そのために、アプリケーションスロットを使用できます。デプロイメントを本番環境で構成できる "pre-production" スロットを設定し、最新のコードをデプロイします。これで、アプリケーションを安全にテストすることができます。アプリケーションに確信を持てたら、本番環境にスワップします。スワップ操作はアプリケーションを再起動せず、代わりに Controller が FrontEnd ロードバランサにトラフィックを最新のスロットにリダイレクトするようにします。

一部のアプリケーションは、本番環境でのロードを安全に処理する前にウォームアップをする必要があります。例えば、アプリケーションでデータのキャッシュにロードする必要がある場合や、.NET アプリケーションで .NET ランタイムがアセンブリを JIT できるようにする必要がある場合などです。この場合、アプリケーションスロットを使用して、アプリケーションを本番環境にスワップする前にウォームアップすることもできます。

アプリケーションのテストとウォームアップの両方に使用される pre-production スロットを持っているお客様をよく目にします。Visual Studio Release Manager などの継続的デプロイツールを使用して、コードを pre-productionスロットにデプロイするためのパイプラインを設定し、検証のためのテストを実行し、アプリを本番環境にスワップする前に必要な全ての URL パスをウォームアップできます。


## スケールユニットのネットワーク構成

App Service スケールユニットは、Could Service 上にてデプロイされております。そのため、アプリケーションへのネットワークの効果を完全に理解するために使い慣れたネットワーク構成と機能となります。

スケールユニットは、世界に公開されている単一の仮想 IP（VIP）アドレスを持っています。スケールユニット単位に割り当てられたすべてのアプリケーションはこの VIP を通じてサービスを提供しております。VIP は、App Service スケールユニットがデプロイされている Cloud Service です。

App Service アプリケーションは、HTTP（ポート 80）と HTTPS（ポート 443）トラフィックのみを提供しております。各 App Service アプリケーションは、azurewebsites.net ドメインに対するデフォルトの組み込み HTTPS がサポートされています。App Service は、Server Name Indication（SNI）証明書と IP ベースの Secure Sockets Layer （SSL）証明書の両方をサポートしています。IP ベースの SSL の場合、アプリケーションには、Cloud Service へデプロイされ関連付けられている受信専用の IP アドレスが割り当てられます。注意：FrontEnd は、すべてのタイプの証明書に対するアプリケーションの HTTPS 要求の SSL 接続を終了します。そして、FrontEnd は、指定されたアプリケーションの Worker へリクエストを転送します。


## パブリック VIP

デフォルトでは、すべての受信 HTTP トラフィックに対して、単一のパブリック VIP があります。すべてのアプリケーションは、単一の VIP にアドレスを指定できます。App Service 上にアプリケーションがある場合は、（Windows または、PowerShell コンソールから）nslookup コマンド実行して結果を確認します。次に例を示します。

```XML
#1 PS C:\> nslookup awesomewebapp.azurewebsites.net
#2 Server:  UnKnown
#3 Address:  10.221.0.3
#4 Non-authoritative answer:
#5 Name:    waws-prod-bay-001.cloudapp.net
#6 Address:  168.62.20.37
#7 Aliases:  awesomewebapp.azurewebsites.net
```

awesomewebapp.azurewebsites.net の出力の調査は以下の通りです。

* #1 行目は、awseomwebapp.azurewebsites.net の nslookup クエリ解決を実行します。
* #5 行目は、awseomwebapp アプリを実行しているスケールユニットのドメイン名を示しています。App Service スケールユニットが Azure Cloud Service によって（cloudapp.net サフィックス）デプロイされていることが分かります。WAWS は、Windows Azure Web Site の略です。（Azure がまだ Windows と呼ばれていたときに、App Service の元の名前です。）
* #6 行目は、スケールユニットの VIP を示しております。waws-prod-bay-001（＃5 行目）でホストおよび実行されているすべてのアプリケーションは、指定されたパブリック VIP でアドレス指定できます。
* #7 行目は、同じ IP アドレスにマッピングされたドメインエイリアスを示しています。


## アウトバウンド VIP

ほとんどの場合、アプリケーションは他の Azure や Azure 以外のサービスに接続されています。そのため、アプリケーションは、スケールユニットではないエンドポイントに対してアウトバウンドネットワークを呼び出します。これには、SQL データベースや Azure ストレージなどの Azure サービスへの呼び出しが含まれています。アウトバウンド通信には、最大 5 つの VIP（1 つのパブリック VIP と 4 つのアウトバウンド専用の VIP）が使用されます。アプリケーションが使用する VIP を選択することはできず、スケールユニット内のすべてのアプリケーションのすべてのアウトバウンド呼び出しは、割り当てられた 5 つの VIP を使用します。アプリケーションにて、API 呼び出しを許可する IP をホワイトリストに登録するサービスを使用している場合には、スケールユニットの 5 つの VIP 全てを登録する必要があります。図 3 に示すように、スケールユニット単位（または、アプリケーション観点から）のアウトバウンド VIP に割り当てられている IP を表示するには、Azure ポータルにアクセスします。

![image-47f2a471-9775-48eb-91e4-e068406ef5bd.png]({{site.baseurl}}/media/2023/05/image-47f2a471-9775-48eb-91e4-e068406ef5bd.png)

図 3: Azure Portal にて、App Service アプリケーションのアウトバウンド IP アドレスを確認

インバウンド IP とアウトバウンド IP の専用のセットを探している場合には、ネットワークから完全に分離された専用の App Service Environment を使用して調べることができます。（ https://bit.ly/2VKdaog ） 


## IP と SNI SSL

App Service は、IP ベースの SSL 証明書をサポートしています。IP-SSL を使用する場合、App Service はインバウンド HTTP トラフィック専用の IP アドレスをアプリケーションに割り当てます。

その他の Azure 専用 IP アドレスとは異なり、IP-SSL を介した App Service の IP アドレスは、使用することを選択した場合のみに割り当てられます。IP アドレスは所有していないため、IP-SSL を削除すると IP アドレスが失われる可能性があります。（他のアプリケーションに割り当てられる可能性があるため）

また、App Service は、SNI SSL もサポートしており、専用の IP アドレスを必要とせず、最新のブラウザでサポートされております。

## アウトバウンドネットワーク呼び出しのためのネットワークポートキャパシティ

アプリケーションの一般的な要件として、他のネットワークエンドポイントへのネットワーク呼び出しを行う性能です。これには、SQL データベースや Azure ストレージなどの Azure 内部サービスへの呼び出しが含まれます。また、アプリケーションが HTTP / HTTPS API エンドポイントを呼び出す場合も含まれます。例えば、Bing Search API を呼び出したり、Web アプリケーションのバックエンドのビジネスロジックを実装している API "アプリケーション" を呼び出したりもします。

これらのほとんどの場合、App Service で実行されている呼び出し元のアプリケーションは、暗黙的にネットワークソケットを開き、Azure ネットワークの観点から、"リモート" とみなすエンドポイントへアウトバウンド呼び出しを実行します。これは重要なポイントで、Azure App Service 上で実行されているアプリケーションから、リモートエンドポイントの呼び出しは、Azure Networking に依存して、ネットワークアドレス変換（NAT）マッピングのテーブルを設定と管理をしております。

NAT マッピングで新しいエントリを作成するには時間がかかり、単一の Azure App Service スケールユニットに対して確立できる NAT マッピングの総数に制限がございます。このため、App Service は、ある時点で未処理になる可能性のあるアウトバウンド接続数制限がございます。

最大接続数制限は以下の通りです。

* B1/S1/P1 インスタンスあたり 1,920 の接続数
* B2/S2/P2 インスタンスあたり 3,968 の接続数
* B3/S3/P3 インスタンスあたり 8,064 の接続数
* I1/I2/I3 インスタンスあたり 16,000 の接続数

接続が "リーク" するアプリケーションは、常にこれらの接続数の制限に達します。リモートエンドポイントへの呼び出しが失敗するため、アプリケーションも断続的に失敗します。その障害は、アプリケーションの負荷が高い期間と密接に関連している場合があります。次のようなエラーが発生しております。"An attempt was made to access a socket in a way forbidden by its access permissions aaa.bbb.ccc.ddd."

この問題が発生する可能性は、以下のようなベストプラクティスで大幅に軽減できます。

* ADO.NET/EF を使用する .NET アプリケーションの場合、データベース接続プールを使用します。
* php/MySQL の場合、永続的なデータベース接続を使用します。
* アウトバウンドの HTTP/HTTPS 呼び出しを行う Node.js アプリケーションの場合、アウトバウンド接続が再利用されるように Keep Alive を構成します。（ https://bit.ly/3mM6mCs ）
* アウトバウンド HTTP/HTTPS 呼び出しを行う .NET アプリケーションの場合、System.Net.Http.HttpClient のインスタンスをプールして再利用するか、System.Net.HttpWebRequest にて KeepAlive 接続を使用します。注意：System.Net.ServicePointManager.DefaultConnectionLimit の値を増やすことを心かげてください。値を増やさない場合、同じエンドポイントへ 2 つのアウトバウンド接続に制限されてしまいます。

App Service SandBox による追加の制限があります。これらは、Azure App Service の低レベルの制限と制約であり、詳細はこちらで確認できます。
 https://bit.ly/2hXJ6lL


## スケールユニットコンポーネント

スケールユニットのコンポーネントは相互に大きく依存しているように見えます。しかし、設計上はこれらは疎結合となっております。Web Worker で現在実行されている（HTTPトラフィックをアクティブに処理している）アプリケーションは、スケールユニット内の他の Role が機能していない場合でも、継続して HTTP トラフィックを処理できます。

例えば、Publisher が適切に機能していないと FTP アクセスが妨げられる可能性がありますが、アプリケーションの HTTP トラフィックやその他のデプロイオプションには影響しません。Controller に新しいアプリケーションの作成を妨げる問題が発生しても、スケールユニットにすぐ割り当てられるアプリケーションが機能しなくなるわけではございません。


## まとめ

Azure App Service は、Web、モバイルや API アプリケーション向けの豊富な PaaS を提供しています。サービスの多くは内部的な部分で、これらは抽象化されているため、開発者は優れたアプリケーションの作成に集中でき、サービスはグローバル規模で実行する複雑さも処理します。

App Service のベストプラクティスの多くは、アプリケーションのスケーリングを中心に発展しております。最適なスケーリング構成を維持しながら、サービスで実行するアプリケーションの数を増やすには、App Service Plan にてアプリケーションが Web Worker へどのようにマッピングされるかを理解することが重要です。

私たちが運用しているクラウドファーストの世界を考えると、Azure と Azure App Service は常に急速に進化しています。2017 年には多くのイノベーションが期待されています。


<br>
<br>

---

<br>
<br>

2023 年 05 月 10 日時点の内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。

<br>
<br>