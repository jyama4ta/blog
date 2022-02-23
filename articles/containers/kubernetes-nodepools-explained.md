---
title: Kubernetes のノードプールについて
date: 2022-02-18 12:00
tags:
  - Containers
  - Azure Kubernetes Service (AKS)
---

こんにちは、Azure テクニカル サポート チームの山下です。
今回は、よくお問い合わせをいただく AKS (Kubernetes) のノードプールについて、2021 年 7 月 12 日に米国の [Core Infrastructure and Security Blog](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/bg-p/CoreInfrastructureandSecurityBlog) で公開された [Kubernetes Nodepools Explained](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/kubernetes-nodepools-explained/ba-p/2531581) を意訳して紹介いたします。

<!-- more -->

## はじめに
この記事では Kubernetes においてノードプールが使用されるユースケースについて、紹介いたします。

1. ノードプールとは何か？
1. システムノードプール、ユーザーノードプールとは？
1. Label と nodeSelector を使用して、どのようにアプリケーションの Pod がノードプールにスケジュールされるのか？
1. Taint と Tolerations を使用することで、どのように特定のアプリケーションだけがスケジュールされるのか？
1. クラスターのアップグレードに伴うリスクを低減するため、どのようにノードプールを使用するか？
1. 各ノードプールの自動スケールをどのように設定するか？

Kubernetes のクラスターでは、コンテナは Pod として Worker Node と呼ばれる VM へデプロイされます。