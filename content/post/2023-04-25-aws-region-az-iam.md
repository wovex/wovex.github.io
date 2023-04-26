---
title: AWS筆記(1) - Region & AZ & IAM
subtitle: 
date: 2023-04-25 12:00:00
tags: ["Ultimate AWS Certified Developer Associate"]
---


# 前言

{{< line_break >}}

在 Udemy 趁優惠買了這堂 AWS 課程 [Ultimate AWS Certified Developer Associate 2023 NEW DVA-C02](https://www.udemy.com/course/aws-certified-developer-associate-dva-c01/)，對於 AWS 我也不算是不熟，只是過去都是需要什麼就看一下那個服務怎麼使用，這次買課程就是想要有規劃的好好學習 AWS


其實在買課程我很是猶豫，因為這位講師在 Udemy 開了不少 AWS 相關的課程，看了課程內容章節，感覺有不少重疊的部分，這門跟這門的差別在哪，老實說看的頭很昏，就學生人數來看，其實比較多人買的是這門課 [Ultimate AWS Certified Solutions Architect Associate SAA-C03](https://www.udemy.com/course/aws-certified-solutions-architect-associate-saa-c03/)，但我最後我還是選擇了這門 Ultimate AWS Certified Developer Associate，理由沒什麼特別的，只是因為這門的講座數比較多看起來比較划算！ (好吧，我承認我放棄比較內容差別了 )


總之，之後我會寫些筆記記錄上述 AWS 課程的學習內容

<!--more-->

{{< line_break >}}


# AWS全球基礎設施

{{< line_break >}}

AWS全球基礎設施目前有
- 31 區域 (Regions)
- 99 個可用區域 (Availability Zones)
- 400+ 個邊緣站點 (Edge Locations) 和 13 個區域邊緣快取 (Regional Edge Caches)

詳細位置可以在這個網站查看：https://infrastructure.aws/

{{< line_break >}}

## AWS Regions

{{< line_break >}}

- AWS Region 遍布全球
- 是一個資料中心叢集
- 大部分的 AWS 服務都是 region-scoped ，會依區域分開

{{< line_break >}}

如何選擇 Regions？
- Compliance 合規：考量當地政策，有些政府會規定資料只能部屬在當地
- Proximity to customers：是否地理上離主要客戶近，以降低 latency
- Available Service：有些服務只有在特定幾個 Region 才有


{{< line_break >}}

## AWS Availability Zones (AZ)

{{< line_break >}}

- 每個 Region 內會有複數個 AZ
- 是 AWS Region 中一個或多個擁有冗餘電源、網路和連接性的獨立資料中心
- AZ 彼此間分開獨立，如此才能達到災難隔離
- Region 中的所有 AZ 彼此以高輸送量、低延遲的網路連接


{{< line_break >}}

## AWS Points of Presense (Edge Locations)

{{< line_break >}}

- 數量最多
- 盡可能地降低資料到客戶端的延遲

{{< line_break >}}

# AWS Identity and Access Management (IAM)

{{< line_break >}}

- 藉由 IAM，可以指定誰或者哪些服務可以存取 AWS 中的服務和資源
- IAM 為 Global Service，不分 Region
- IAM是免費的服務
- Group 只能含 User，不能含其他 Group
- Group 方便我們一次設定多個 User
- User 可以屬於多個 Group，也可不屬於任何 Group

{{< line_break >}}

## IAM Policy

{{< line_break >}}

- JSON 檔案
- 用來設定 User 或 Group 的哪些操作可以被使用在哪些 AWS 資源上


{{< line_break >}}

### IAM Policy 檔案結構

{{< line_break >}}

- Version：policy language version, always include “2012-10-17”
- Id：an identifier for the policy (可選)
- Statement：one or more individual statements (必填)
    - Sid：an id for the statement (可選)
    - Effect：whether the statement allows or denies access (Allow or Deny)
    - Principal：account/user/role to which this policy applied to (主詞)
    - Action：list of actions this policy allows or denies (動作)
    - Resource：list of resources to which the actions applied to (資源受詞)
    - Condition：conditions for when this policy is in effect (可選) (條件)

{{< line_break >}}

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:GetBucketPolicy",
            "Resource":"arn:aws:s3:::bucket-name/*"
        }
    ]
}
```

{{< line_break >}}

## IAM Role for Services

{{< line_break >}}

我們可以設定 IAM Roles 到一些 AWS Service，讓該 Service 有能力存取 AWS 資源，像是：
- EC2 Instance Roles
    - 每一個 EC2 instance 只能與一個 IAM Role 的綁定
- Lambda Function Roles
- Roles for CloudFormations

{{< line_break >}}

IAM Role 有以下三種：

1. AWS Service Role
2. Role for Cross-Account Access：用來設定跨帳號存取權限用
3. Role for Identity Provider Access：外部帳號(不是 IAM User )需要存取權限，就必須透過整合 SAML provider (例如：AD)的方式，搭配 IAM Role 來取得所需要的權限

{{< line_break >}}

## IAM API Keys

{{< line_break >}}

- API Access Key 必須與 IAM User 綁定，來取得對應的存取權限
- 使用 API Key 省去輸入帳密動作
- 千萬不要把 API Access Key, Secret Access Key 放到 EC2 中 (應使用 IAM Role )


{{< line_break >}}

## IAM Security Tools

{{< line_break >}}


AWS提供一些工具幫助我們管理IAM帳號

- IAM Credentials Report (account-level)
    - User列表資料
- IAM Access Advisor (user-level)
    - 某 User 的 Service 權限，與其上次使用該 Service 的時間


{{< line_break >}}

## IAM Guidelines & Best Practices

{{< line_break >}}


- 別使用 root account，除了在 AWS account 建立初期
- 一個實體用戶只會對應一個 AWS User
- 將 User 歸在 Group 內，針對 Group 做權限設定
- 使用 Strong password policy
- 使用 MFA
- 使用 IAM Role 賦予 Service 權限
- CLI/SDK 使用 Access Key
- 使用 IAM Security Tools 規劃權限設定
- 永遠別分享你的 IAM User 和 Access Key

{{< line_break >}}
