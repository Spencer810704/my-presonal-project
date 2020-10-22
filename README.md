# Progress

---

持續更新內容中.....

- [x]  架構圖
- [ ]  服務安裝筆記
- [ ]  代碼筆記

# Introduction

---

## Architecture

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3b61ca2a-1ddd-402e-9696-ad11aec027fc/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3b61ca2a-1ddd-402e-9696-ad11aec027fc/Untitled.png)

近年資訊業的蓬勃發展下，很多工具或是雲平台如雨後春筍般的出現，像是GCP、Azure、阿里雲、騰訊雲等等平台或像是Ansible、Terraform、SaltStack這種Infrastructure As Code的配置管理工具，而每一家公司至少也都會使用一至多個雲平台，並且當一個雲平台下的帳號不只一個，帳號管理數量一多，常常會發生一些奇葩的事情，項是帳號密碼被某個工程師修改了，但是卻沒有讓到團隊成員知道，或者是某個服務器遷移至某個帳號下管理，團隊內部資訊同樣沒有同步，造成其他成員嘗試多個帳號後才發現已經遷移，而有些團隊很多時候都在這種事情上浪費人力，而這種情況需要被解決，透過一些文章了解ITIL裡面提到的CMDB概念 : 配置管理資料庫(CMDB)是與IT系統所有組件相關的信息庫。它包含IT基礎架構配置項的詳細信息，才衍生出了這個想法，讓團隊成員能夠透過CMDB集中化管理資產訊息、配置等等，並且結合常見的Devops Tools整合所有雲平台內容資產訊息，而團隊成員只需要專注在CMDB上進行機器或者配置的新增、刪除、修改、更新，而不需要再耗費其他時間去熟悉雲平台配置，讓其他成員更有時間去專注在他們的專業領域上。

## Component

使用原因

- Terraform : 透過Terraform將雲平台的API封裝，不需要了解API的實際操作方式就能對雲平台操作(新增、刪除、Blabla)，未來當平台切換時也能夠抽換到對應雲平台的Provider。
- Proxmox VE : 恩...因為我沒錢(ಥ﹏ಥ)，剛好這套免費虛擬化技術安裝簡易Terraform又有支援。
- CMDB : 想透過做中學的方式學習Python Django框架。
- Jenkins : 將每一次Ansible執行結果紀錄 ( 想讓CMDB專職管理資產等等內容，而一些特殊客製化內容透過Jenkins pipeline再進行定義 )。
- Ansible : Terraform作為機器初始化的配置管理，而Ansible作為業務層面的配置管理(例如安裝GO、Java、Python)，ython)，另外個人較偏好於使用Agentless方式管理服務器，並且Ansible Galaxy上也有提供許多的Role能夠使用，秉持著Stop Trying to Reinvent the Wheel的精神。
- Zabbix : 服務器建立後可以監控相關資源，透過Ansible配置，一方面也是能夠學習監控領域。