---
title: "針對 Ansible Tower 升級 AAP 2 建議方案及執行確認"
slug:  "ansible"
url: "ansible/ansible-tower-upgrade-to-ansible-automation-platform-2"
date: 2022-12-19T12:00:40+08:00
draft: false
description: "Ansible Tower 產品生命週期演進關係，企業會開始需要進行 AAP 2 升級計畫，本篇文會介紹如何將現有 AAP 1.x (Ansible Tower 3.X)，包含當前部署模式、腳本工作流程方式及遷移相關等的任何複雜性評估，並介紹 Automation execution environments 解決了什麼問題，後續如何透過 AAP2 建構我們的自動化中台需求。"

categories:
  - Ansible Automation Platform
tags:
  - Ansible Automation Platform
---
![](https://i.imgur.com/dVmSATw.png)


♖ Ansible Tower 的升級路線
Ansible Tower 升級可以參考 Upgrading an Existing Tower Installation，並且分為以下兩種情況：

(1) 假設您的 Ansible Tower 版本在 3.7 以上基本上可以總結為以下三步驟：

使用官方下載連結下載目標版本 Ansible Tower 壓縮檔並解壓縮
備份舊版本的 Ansible Tower 資料（包含 Templates, Projects 等設定），備份步驟可以參考 Backing Up and Restoring Tower
複製備份檔案到目標版本的 Ansible Tower 目錄下
執行目標版本的安裝流程
還原備份檔案
(2) 假設您的版本在 3.7 以下，則可以參考 Red Hat Ansible Tower Upgrade from 3.5 to 3.8 – when running setup.sh is not enough – or: I have made fire!

最保險的升級建議是升級到目前版本的最新小版本，再升級到下一個中版本，比方說目前版本是 3.7.2，就建議升級到 3.7.5 (3.7系列的最後一版) 再往上升到 3.8.4。

- 參考 Migration from AAP 1.2 to Ansible Automation Platform 2: Side by side upgrade - Step 1 指南參考：

{{< youtube EKf3u1QdpNo >}}


The first video in a five part series offering a step by step walkthrough of a side by side migration from Ansible Automation Platform 1.2 to Ansible Automation Platform 2. 

Checklist | 5 ways to prepare for migration to Ansible Automation Platform 2: https://www.redhat.com/en/resources/prepare-for-migration-ansible-automation-2-checklist