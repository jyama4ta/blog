---
title: マネージド ディスクのクレジットベースのバーストについて
date: 2021-12-13 09:00:00
tags:
  - VM
  - Disk
---

こんにちは、Azure テクニカル サポート チームの山下です。

今回は、マネージド ディスクのクレジットベースのバーストの挙動に関して、弊社公開ドキュメントの補足、ならびに実際の動作について、検証結果も交えてご紹介いたします。

<!-- more -->

## はじめに
Azure のディスクでは、ディスクのサイズに応じて性能（スループット、IOPS）が設定されています。ディスクの性能を向上されたい場合には、ディスクのサイズ拡張を行うという、シンプルな仕組みで提供されております。しかしながら、ディスクの拡張は容易に行える一方で、一度拡張したディスクは縮小が行えないという制限事項があるため、仮想マシンの起動時や短時間のバッチ処理などで一時的に向上されたい場合には、拡張いただくのが難しいという課題がございました。
今回ご紹介するクレジットベースのバーストは、ディスクレベル・仮想マシンレベルそれぞれで追加コストなく、短時間の性能向上を提供する機能となります。

同種の機能として、今回ご紹介するクレジットベースのバーストの他にも、オンデマンド バーストと呼ばれる追加コストが発生する代わりに時間制限なくバーストできる機能や、継続的に上位の性能に変更するパフォーマンス レベルの変更といった機能もございます。それぞれの公開ドキュメントを以下に記載しておりますので、詳細はそちらをご参照いただけますと幸いです。また、パフォーマンス レベルの変更に関しましては、以前に当ブログでも [紹介](https://jpaztech.github.io/blog/vm/introduction-performance-tier-ssd/) しておりますので、こちらもご参考となれば幸いです。


> マネージド ディスクのバースト  
> [https://docs.microsoft.com/ja-jp/azure/virtual-machines/disk-bursting](https://docs.microsoft.com/ja-jp/azure/virtual-machines/disk-bursting)  
> オンデマンドのバーストを有効にする  
> [https://docs.microsoft.com/ja-jp/azure/virtual-machines/disks-enable-bursting](https://docs.microsoft.com/ja-jp/azure/virtual-machines/disks-enable-bursting)  
> マネージド ディスクのパフォーマンス レベル  
> [https://docs.microsoft.com/ja-jp/azure/virtual-machines/disks-change-performance](https://docs.microsoft.com/ja-jp/azure/virtual-machines/disks-change-performance)  


## クレジットベースのバーストの特徴
[公開ドキュメント](https://docs.microsoft.com/ja-jp/azure/virtual-machines/disk-bursting) より、上述の各バーストの特徴を記載した表を以下に抜粋いたします。

| | クレジットベースのバースト | オンデマンド バースト | パフォーマンス レベルの変更 |
|--|--|--|--|
| シナリオ | 短期的なスケーリング (30 分以下) に最適です。 | 短期的なスケーリング (時間制限なし) に最適です。 | ワークロードが継続的にバーストで実行される場合に最適です。|
| コスト | Free | コストは変動します。詳細については、[課金](https://docs.microsoft.com/ja-jp/azure/virtual-machines/disk-bursting#billing)に関するセクションをご覧ください。 | 各パフォーマンス レベルのコストは固定です。詳細については、「[Managed Disks の価格](https://azure.microsoft.com/pricing/details/managed-disks/)」をご覧ください。|
| 可用性 | 512 GiB 以下の Premium および Standard SSD でのみ使用できます。 | 512 GiB を超える Premium SSD でのみ使用できます。 | すべての Premium SSD サイズで使用できます。 |
| 有効化 | 対象のディスクでは既定で有効になっています。 | ユーザーが有効にする必要があります。 | ユーザーが手動でレベルを変更する必要があります。|

表に記載の通り、クレジットベースのバーストは、お客様側で何らかの操作をすることなく、追加コスト無しに性能が向上する点が大きなメリットとなります。一方で、対象となるディスクサイズに制限があることや、バーストされる時間が最大でも 30 分という制約もございます。既に 512 GiB よりも大きいディスクをご利用されている場合や、30 分以上のバッチ処理のために性能向上を目的とされる場合には、オンデマンド バーストやパフォーマンス レベルの変更を検討いただく必要がある点はご注意いただければと存じます。

また、上記はディスクレベルのバーストとなりますが、[仮想マシンレベルのバースト](https://docs.microsoft.com/ja-jp/azure/virtual-machines/disk-bursting#virtual-machine-level-bursting)もございます。仮想マシンレベルのバーストは、ディスクとは異なり、クレジットベースのバーストのみが提供されており、ご利用いただけるサイズも以下のサイズのみとなっております。ディスクレベルのバースト

- [Dsv4 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/dv4-dsv4-series)
- [Dasv4 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/dav4-dasv4-series)
- [Ddsv4 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/ddv4-ddsv4-series)
- [Dasv5 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/dasv5-dadsv5-series)
- [Dadsv5 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/dasv5-dadsv5-series)
- [Esv4 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/ev4-esv4-series)
- [Easv4 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/eav4-easv4-series)
- [Edsv4 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/edv4-edsv4-series)
- [Easv5 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/easv5-eadsv5-series)
- [Eadsv5 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/easv5-eadsv5-series)
- [B シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/sizes-b-series-burstable)
- [Fsv2 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/fsv2-series)
- [Dsv3 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/dv3-dsv3-series)
- [Esv3 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/ev3-esv3-series)
- [Lsv2 シリーズ](https://docs.microsoft.com/ja-jp/azure/virtual-machines/lsv2-series)


## 使用方法
ここでは Azure Portal からの設定方法、ならびに実際にパフォーマンスが向上している様子をご紹介いたします。Azure Portal だけではなく、Azure PowerShell, Azure CLI からも設定が行えますので、CUI で変更されたい場合には、公開ドキュメントのご参照をお願いいたします。

今回は以下のような環境で、データ ディスクにおけるパフォーマンス レベルの変更を確認します。

- VM サイズ：Standard D16s_v3
- ディスク サイズ：P4 (120 IOPS / 25 MBps)
- 変更後のパフォーマンス レベル：P40 (7500 IOPS / 250 MBps)
- ゲスト OS：Windows Server 2019

### パフォーマンス レベル変更前の性能を確認
iometer を使用して P4 のデータディスクのスループットを計測した結果、以下の画像のように、P4 の上限値である 25 MBps 前後の性能であることが確認できます。
![](./introduction-performance-tier-ssd/originalthroughput.png)

### パフォーマンス レベルの変更
1. VM の停止操作を行い、割り当て解除の状態にします。
![](./introduction-performance-tier-ssd/deallocated.png)
1. パフォーマンス レベルを変更されたいディスクを開き、[設定] - [サイズおよびパフォーマンス] にアクセスします。
1. 下部にある [パフォーマンス レベル] のプルダウンより、変更されたいパフォーマンス レベルを選択します。ここでは P40 を選択し、[サイズ変更] をクリックします。
![](./introduction-performance-tier-ssd/performancelevel.png)
1. 更新が完了したメッセージが表示されるのを待ちます。
![](./introduction-performance-tier-ssd/changed.png)
1. 更新完了後、VM を開始します。

### パフォーマンス レベル変更後の性能を確認
再度 iometer を使用してスループットを計測した結果が以下です。スループットが先ほどの 25 MBps から、250 MBps 前後まで向上していることが確認できます。
![](./introduction-performance-tier-ssd/highthroughput.png)

### よくあるご質問




本稿が少しでも皆様のご参考となれば幸いです。
