# SentinelUploadMDEIndicatorV2

Microsoft Sentinel のインシデント発生時に、関連する IP アドレスを **Microsoft Defender for Endpoint (MDE)** の IoC (Indicator of Compromise) として自動登録する Logic App Playbook です。

## 概要

```
Sentinel インシデント発生
        │
        ▼
  IP エンティティ取得
        │
        ▼
  IP アドレスが存在するか？ ── No ──▶ 終了
        │
       Yes
        ▼
  各 IP をループ処理
        │
        ▼
  プライベート IP を除外
  (10.x / 172.16.x / 192.168.x)
        │
        ▼
  MDE API で IoC 登録
  (AlertAndBlock / High / 30日有効)
        │
        ▼
  Sentinel インシデントにコメント追加
```

## 機能

| 項目 | 内容 |
|---|---|
| **トリガー** | Microsoft Sentinel インシデント作成 |
| **対象エンティティ** | IP アドレス |
| **IoC アクション** | `AlertAndBlock`（アラート発報 & 通信ブロック） |
| **重大度** | High |
| **有効期限** | 登録日から 30 日間 |
| **除外対象** | プライベート IP（RFC 1918: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`） |
| **コメント** | 登録成功時に Sentinel インシデントへ自動コメント |

## 前提条件

- Azure サブスクリプション
- Microsoft Sentinel ワークスペース（有効化済み）
- Microsoft Defender for Endpoint ライセンス
- Azure AD アプリ登録（MDE API アクセス用）
  - API アクセス許可: `Ti.ReadWrite`（Microsoft Defender for Endpoint）

## パラメーター

| パラメーター | 説明 | 既定値 |
|---|---|---|
| `PlayBookName` | Logic App のリソース名 | `SentinelUploadMDEIndicator` |
| `AadTenantId` | Azure AD テナント ID | — |
| `AadClientId` | アプリ登録のクライアント ID | — |
| `AadClientSecret` | アプリ登録のクライアント シークレット | — |

## デプロイ

### Azure Portal からデプロイ

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisashin0728%2FSentinelUploadMDEIndicator%2Frefs%2Fheads%2Fmain%2FSentinelUploadMDEIndicatorV2.json)

### Azure CLI からデプロイ

```bash
az deployment group create \
  --resource-group <リソースグループ名> \
  --template-file SentinelUploadMDEIndicatorV2.json \
  --parameters \
    PlayBookName="SentinelUploadMDEIndicator" \
    AadTenantId="<テナントID>" \
    AadClientId="<クライアントID>" \
    AadClientSecret="<クライアントシークレット>"
```

### PowerShell からデプロイ

```powershell
New-AzResourceGroupDeployment `
  -ResourceGroupName "<リソースグループ名>" `
  -TemplateFile "SentinelUploadMDEIndicatorV2.json" `
  -PlayBookName "SentinelUploadMDEIndicator" `
  -AadTenantId "<テナントID>" `
  -AadClientId "<クライアントID>" `
  -AadClientSecret "<クライアントシークレット>"
```

## デプロイ後の設定

1. **マネージド ID へのロール割り当て**
   - Logic App のシステム割り当てマネージド ID に対し、Sentinel ワークスペースの **Microsoft Sentinel Responder** ロールを付与してください。

2. **API 接続の認可**
   - Azure Portal で Logic App を開き、`azuresentinel` API 接続の認可を完了してください。

3. **Sentinel オートメーション ルール（任意）**
   - 特定の条件でのみ Playbook を実行する場合は、オートメーション ルールを構成してください。

## アーキテクチャ

```
┌──────────────────────┐
│  Microsoft Sentinel  │
│   (Incident Trigger) │
└──────────┬───────────┘
           │ Webhook
           ▼
┌──────────────────────┐     REST API      ┌──────────────────────┐
│     Logic App        │ ───────────────▶  │  Microsoft Defender  │
│  (Playbook)          │   IoC 登録        │  for Endpoint        │
│                      │                   │  (Ti.ReadWrite)      │
│  - IP エンティティ取得 │                  └──────────────────────┘
│  - プライベート IP 除外│
│  - IoC 登録           │
│  - コメント追加       │
└──────────────────────┘
```

## 認証方式

| 接続先 | 認証方式 |
|---|---|
| Microsoft Sentinel | マネージド ID (システム割り当て) |
| Microsoft Defender for Endpoint API | Azure AD OAuth 2.0 (クライアント資格情報) |

## ライセンス

MIT License
