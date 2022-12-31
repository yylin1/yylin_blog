---
title: "針對 Ansible Tower 升級 AAP 2 建議方案及執行確認"
slug:  "ansible"
url: "ansible/ansible-tower-upgrade-to-ansible-automation-platform-2"
date: 2022-12-19T12:00:40+08:00
draft: true
description: "Ansible Tower 產品生命週期演進關係，企業會開始需要進行 AAP 2 升級計畫，本篇文會介紹如何將現有 AAP 1.x (Ansible Tower 3.X)，包含當前部署模式、腳本工作流程方式及遷移相關等的任何複雜性評估，並介紹 Automation execution environments 解決了什麼問題，後續如何透過 AAP2 建構我們的自動化中台需求。"
categories:
  - Ansible Automation Platform
tags:
  - Ansible Automation Platform
---

隨著 Ansible Automation Platform 2.0 於今年七月中釋出 Early Access 版本，我之後會寫一篇文章介紹與大家之前比較熟悉的 Ansible Tower 在特性上有什麼差別，有興趣的朋友可以先在新的環境玩玩看，先不要急著升級，因為之後會有更多新功能，也會在之後的文章一併介紹。

本篇主要是介紹升級 Ansible Tower 3.8.4 (Ansible Tower 的最後一版) 到 Ansible Automation Platform 2.0 的作法。

之所以會有這篇文章的出現，主要是因為 Ansible Automation Platform 2.0 以及之後的版本對於 RHEL 作業系統和 Ansible-core 的版本都會有一些限制，所以先寫篇文章記錄一下，之後有朋友想升級也可以當個參考 。"

## /環境/
RHEL 7.9
RHEL 8.4
Ansible Tower 3.8.4
Ansible Automation Platform 2.0 Early Access
(以下簡稱 AAP2.0)


## /Ansible Tower 的升級路線/

Ansible Tower 升級可以參考 Upgrading an Existing Tower Installation，並且分為以下兩種情況：

(1) 假設您的 Ansible Tower 版本在 3.7 以上基本上可以總結為以下三步驟：

使用官方下載連結下載目標版本 Ansible Tower 壓縮檔並解壓縮
備份舊版本的 Ansible Tower 資料（包含 Templates, Projects 等設定），備份步驟可以參考 Backing Up and Restoring Tower
複製備份檔案到目標版本的 Ansible Tower 目錄下
執行目標版本的安裝流程
還原備份檔案
(2) 假設您的版本在 3.7 以下，則可以參考 Red Hat Ansible Tower Upgrade from 3.5 to 3.8 – when running setup.sh is not enough – or: I have made fire!

最保險的升級建議是升級到目前版本的最新小版本，再升級到下一個中版本，比方說目前版本是 3.7.2，就建議升級到 3.7.5 (3.7系列的最後一版) 再往上升到 3.8.4。

♖ Ansible Tower 到 Ansible Automation Platform 2.0 的升級路線
老實說 Ansible Tower 本身升級過程沒什麼問題，就是 Ansible-core 的版本問題和 AAP2.0 對 RHEL 的版本限制花了點時間。

步驟	Ansible Tower / AAP 版本	作業系統版本	Ansible-core 版本
1	Ansible Tower 3.8.4	RHEL7	Ansible core 2.9.X
2	Ansible Tower 3.8.4	RHEL8	Ansible core 2.9.X
3	Ansible Automation Platform 2.0	RHEL8	Ansible core 2.11.X

這邊如果有實驗精神的朋友想要跟我一樣試著退版從 RHEL7 試驗重裝，也可以參考 How to uninstall Ansible Tower，另外附帶一提，安裝 Ansible Tower 會需要用到兩個 package rhel-7-server-extras-rpms & rhel-server-rhscl-7-rpms，這兩個 package 都會需要安裝 epel-release，可參考以下指令：

```

先講一下我的升級路徑：

- 參考 Migration from AAP 1.2 to Ansible Automation Platform 2: Side by side upgrade - Step 1 指南參考：

{{< youtube EKf3u1QdpNo >}}

## Reference

- [Red Hat Ansible Tower Upgrade from 3.5 to 3.8 – when running setup.sh is not enough](https://www.martinberger.com/2020/12/red-hat-ansible-tower-upgrade-from-3-5-to-3-8-when-running-setup-sh-is-not-enough-or-i-have-made-fire/)
- [How Can I Uninstall Ansible Tower?](https://access.redhat.com/solutions/3107181)
- [Backing Up and Restoring Tower](https://docs.ansible.com/ansible-tower/latest/html/administration/backup_restore.html)
- [Ansible Automation Platform 2 Installation Guide](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.2/html/red_hat_ansible_automation_platform_operator_installation_guide/index)
- [Ansible Tower 升級 Ansible Automation Platform 2.0 作法 by Hazel](https://hazel.style/2021/10/29/Ansible-Tower-upgrade-to-Ansible-Automation-Platform-2/#more)