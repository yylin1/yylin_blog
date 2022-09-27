---
title: "實現 OpenSCAP 自動化邏輯檢查"
slug:  "ansible"
url: "ansible/openscap"
date: 2022-09-25T12:00:40+08:00
draft: false
description: "本篇主要介紹如何結合 Ansible Automtaion Platform 和 Red Hat Satellite 和 OpenSCAP 實現大規模系統自動檢查和合規性邏輯檢查。"
categories:
  - Ansible Automation Platform
tags:
  - Ansible Automation Platform
---

## 概述
SCAP (Security Content Automation Protocol)是一包資安標準的集合。實作 SCAP 的工具有好幾種，其中一個是 OpenSCAP 開源工具，運用 OpenSCAP 可做到全自動的檢查、script/ansible playbook的報告與修復。但「修復」這段不太可能全自動，需要人為判斷要不要修復會不會與公司定義的規則互相衝突，否則完全聽從某些嚴格的 SCAP Rules，派下去之後反而造成維運及開發團隊環境配置衝突，導致相關影響及不便性。

## 透過 OpenSCAP 和 Satellite 實現系統合規檢查

在 Red Hat Satellite 中，包含可以配置 OpenSCAP 項目提供的工具用於實施安全合規性檢查。有關 OpenSCAP 的更多信息，可以參見Guide to the Secure [Configuration of Red Hat Enterprise Linux 7](https://static.open-scap.org/ssg-guides/ssg-rhel7-guide-cui.html#!)
和 [Guide to the Secure Configuration of Red Hat Enterprise Linux 8](http://static.open-scap.org/ssg-guides/ssg-rhel8-guide-cui.html#!)。通過 Satellite Web UI，可以在 Red Hat Satellite 管理的所有主機上進行計劃的合規性檢查和直接查看對應報告。

Red Hat Ansible Automation(簡稱 AAP) 可以協助維運團使 IT 環境和流程的各個方面實現自動化，而 AAP 則可以與其他服務整合。在以下實驗中我們透過影片方式，介紹如何透過 AAP 與 Satellite 來達成OpenSCAP的自動合規邏輯檢查：

執行主要步驟包含：
1. 在 Red Hat Satellite 中定義 OpenSCAP 檢查策略
2. 使用 Red Hat Ansible Automation Platform 於系統主機上大量運行 OpenSCAP 邏輯檢查掃描

> 可以參考實作詳細資訊步驟-參考： [Automated Smart Management Workshop](https://github.com/ansible/workshops/tree/devel/exercises/ansible_smart_mgmt)


{{< youtube QFf0yVAHQKc >}}

## /補充/ OpenSCAP與資安合規生命週期

為什麼需要系統安全檢查強化工具，OpenSCAP 可以完美的參與整個資安合規的生命週期：

- 評估當前系統不合規的地方
- 將找到的問題，依重要性分類
- 修復（需要與企業內部定義規則評估）
- 產生報告

> 重覆以上輪迴直到某主機的安全強度到一可接受的程度)

更棒的是，這四階段可以通通 Ansible Automation Platform 自動化，用來解決下面的資安老問題：

- 做資安檢測費時費工
- 判斷問題、修復問題，容易出錯
- 資安規則太多，很容易漏掉
- 資安規則細如牛毛，要做好不容易
- 人工檢測，即便做成 SOP，也不容易實踐


## Reference
- OpenSCAP on Github: https://github.com/OpenSCAP/openscap
- https://www.open-scap.org/
- https://aap2.demoredhat.com/exercises/ansible_smart_mgmt/
- https://blog.51cto.com/u_15127570/2712917