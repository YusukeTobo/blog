---
title: "Blobコンテナーのコンテナーごとの容量確認方法"
author_name: "Haruka Matsumura"
tags:
    - Storage
---
- 2023/02/21: GUI (Azure ポータル) を用いる方法の機能廃止に伴い、内容を更新いたしました。<br>
- 2024/01/19: Azure PowerShell のスクリプトを用いる方法でご紹介のリンク先のスクリプトが、単一コンテナーのサンプルからストレージ アカウント内の全コンテナー対象のサンプルへの変更に伴い、内容を更新いたしました。<br>
---

# 質問
Blob コンテナーごとの使用状況を把握するために、メトリックより確認できるストレージ アカウントの Blob ストレージ全体の容量ではなく、1 コンテナーずつの容量を確認したいです。一覧表示を行う機能はありますか？<br>
<br>

# 回答
Blob コンテナーごとのサイズは Azure 基盤側で計測しておりませんため、コンテナーごとのサイズを簡易に一覧表示する機能のご用意がございません。<br>
そのため、対象コンテナー内の Blob ファイルのサイズをリストし、これを足し合わせて計算することで確認を行う必要がございます。<br>
<br>
この、「対象コンテナー内の Blob ファイルのサイズをリストし、これを足し合わせて計算することで確認を行う」手段は下記 3 つの方法がございますので、それぞれご紹介いたします。<br>

- [Azure PowerShell のスクリプトを用いる方法](#powershell)
- [GUI (Storage Explorer (デスクトップ アプリケーション版) を用いる方法](#storage-explorer)
- [Azure Storage Blob インベントリ機能と Azure Synapse を用いる方法](#inventory)

<br>
なお、前者 2 つの方法ではコンテナーごとに ListBlobs API を呼び出して、Blob ファイルのサイズをリストいたします。List Blobs API は 1 回の呼び出しでリスト可能な BLOB 数の上限が 5,000 のため、数千万個の BLOB が
コンテナー内に存在した場合、呼び出し回数が多くなります。<br>
この呼び出し回数はストレージの操作件数ごとの課金に影響を及ぼす場合もございますので、あらかじめご留意ください。<br>
<br>

(ご参考) <br>
Blob の一覧表示 (REST API)-Azure Storage<br>
[https://docs.microsoft.com/ja-jp/rest/api/storageservices/list-blobs](https://docs.microsoft.com/ja-jp/rest/api/storageservices/list-blobs)


Azure Storage Blob の価格<br>
[https://azure.microsoft.com/ja-jp/pricing/details/storage/blobs/](https://azure.microsoft.com/ja-jp/pricing/details/storage/blobs/)<br>
([操作とデータ転送] - [List、Create Container 操作 (10,000 件あたり)] に該当します。)<br>
<br>

<a id="powershell"></a>
## Azure PowerShell のスクリプトを用いる方法
PowerShell から Azure リソースを直接管理するためのコマンドレットがセットになった、Azure PowerShell (Az モジュール) をご利用いただき、スクリプトを作り込みいただくことでコンテナーごとのサイズを計算いただくことが可能です。<br>
Azure PowerShell (Az モジュール) のご利用に関しましては、下記公開情報をご参考ください。<br>
<br>
(ご参考) Azure Az PowerShell モジュールの概要<br>
[https://docs.microsoft.com/ja-jp/powershell/azure/new-azureps-module-az?view=azps-5.9.0](https://docs.microsoft.com/ja-jp/powershell/azure/new-azureps-module-az?view=azps-5.9.0)<br>
<br>
スクリプトに関しましては、下記に対象のストレージ アカウント内のすべてのコンテナーのサイズを計算するためのサンプル スクリプトがございますので、お客様にてスクリプトを作り込みいただく際のご参考となれば幸いです。<br>
<br>(ご参考) PowerShell を使用して BLOB コンテナーのサイズを計算する<br>
[https://docs.microsoft.com/ja-jp/azure/storage/scripts/storage-blobs-container-calculate-size-powershell](https://docs.microsoft.com/ja-jp/azure/storage/scripts/storage-blobs-container-calculate-size-powershell)<br>
<br>

<a id="storage-explorer"></a>
## GUI (Storage Explorer (デスクトップ アプリケーション版)) を用いる方法
Storage Explorer の場合も、GUI にて容量を確認したい対象のコンテナーを選択いただき、下記操作をいただくことで対象コンテナ―の計算されたサイズをご確認いただくことが可能です。
<br>
なお、Storage Explorer をご利用いただく場合は、Azure ポータル上の [ストレージ ブラウザー (プレビュー)] / [Storage Explorer (プレビュー)] からはご利用いただけませんので、下記のデスクトップ アプリケーション版をご利用ください。<br>

(ご参考) <br>
Azure Storage Explorer – クラウド ストレージ管理<br>
[https://azure.microsoft.com/ja-jp/features/storage-explorer/](https://azure.microsoft.com/ja-jp/features/storage-explorer/)<br>
<br>
Storage Explorer の概要<br>
[https://docs.microsoft.com/ja-jp/azure/vs-azure-tools-storage-manage-with-storage-explorer?tabs=windows](https://docs.microsoft.com/ja-jp/azure/vs-azure-tools-storage-manage-with-storage-explorer?tabs=windows)<br>
<br>

■ 参考画面<br>
![blob-container-size-check-StorageExplorer.png]({{site.baseurl}}/media/2021/10/2021-10-06-blob-container-size-check-StorageExplorer.png)<br>
<br>

<a id="inventory"></a>
## Azure Storage Blob インベントリ機能と Azure Synapse を用いる方法
Blob とコンテナーのプロパティ情報をレポートとして出力する Blob インベントリ機能と、データ分析を行う Azure Synapse を組み合わせ、出力されたレポートを Azure Synapse で解析することでコンテナーごとのサイズを計算して表示いただくことも可能です。<br>
詳細なご利用方法については、下記に公開情報がございますのでこちらをご参考ください。<br>
<br>
(ご参考) Azure Storage インベントリを使用して Blob の数とサイズを計算する <br>
[https://docs.microsoft.com/ja-jp/azure/storage/blobs/calculate-blob-count-size](https://docs.microsoft.com/ja-jp/azure/storage/blobs/calculate-blob-count-size)<br>
<br>
ただ、こちらの方法は Blob インベントリ機能の追加のコストならびに Azure Synapse リソースの追加のコストが発生いたしますことと、インベントリ レポートの出力に最大 24 時間かかるという点がございます。<br>
このため、散発的な確認用途よりも、レポート情報として毎日の情報を継続的に保存しておきたい、ビッグ データとしての解析用に日々のレポートを出力したいといったご用途に適しているかと思います。<br>
<br>

# 補足
Azure Storage の BLOB は、ブロック BLOB・ページ BLOB・追加 BLOB の三種類がございます。このうち、仮想マシンの OS ディスクに使われる VHD ファイルを格納するページ BLOB は、

- 割り当てられた (プロビジョニングされた) サイズ
- 実際に使用されているサイズ

の二種類のサイズの考え方がございます。このうち、本稿でご紹介した方法で確認できるサイズは、前者の「割り当てられた (プロビジョニングされた) サイズ」となります。<br>

Premium ページ BLOB の場合、課金の指標に利用されるサイズは「割り当てられた (プロビジョニングされた) サイズ」です。一方、Standard ページ BLOB の場合、「実際に使用されているサイズ」が課金の指標であり、本稿でご紹介の方法で確認できるサイズとは差異がございます。<br>

(ご参考) Azure ページ BLOB Storage の価格<br>
[https://azure.microsoft.com/ja-jp/pricing/details/storage/page-blobs/](https://azure.microsoft.com/ja-jp/pricing/details/storage/page-blobs/)<br>

課金料金の推定のために容量を確認される場合で、Standard ページ BLOB をご利用の場合、コンテナー毎の確認はいただけず恐縮ですが、本確認方法ではなく、Azure Monitor のメトリック機能をご活用ください。<br>

■ 参考画面<br>
![page-blob-metric.png]({{site.baseurl}}/media/2021/10/2021-10-06-page-blob-metric.png)<br>
<br>


# まとめ
コンテナーごとの使用容量の確認方法は、上記の通り、いくつか方法がございます。<br>
それぞれ一長一短がございますので、お客様の環境に適した方法をご利用いただければと存じます。<br>
本内容が、少しでも皆様のお役に立てましたら幸いです。

<br>
<br>

---

<br>
<br>

本記事は 2021 年 09 月 30 日時点に初校した内容となります。<br>
本記事の内容は予告なく変更される場合がございますので予めご了承ください。<br>
変更点については、記事先頭に記載しておりますので、ご確認ください。

<br>
<br>
