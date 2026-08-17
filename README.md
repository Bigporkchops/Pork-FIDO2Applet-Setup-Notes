> [!WARNING]
> **硬體相容性提示**：請避免使用 **InfoThink IT-102MU** 讀卡機進行以下操作。

---

### 第一階段：生成私有管理金鑰與安裝參數

在終端機中透過 Python 的 `secrets` 模組生成 3 組具備密碼學安全強度的獨立隨機金鑰：

```bash
python -c "import secrets; print('NEW_ENC:', secrets.token_hex(16).upper()); print('NEW_MAC:', secrets.token_hex(16).upper()); print('NEW_DEK:', secrets.token_hex(16).upper())"

```

> **注意**：產出的 3 組 32 位元 16 進位（Hex）金鑰請妥善備份，未來管理卡片時皆需使用。

---

### 第二階段：生成微軟與 Google 相容的最佳化參數

執行腳本生成安裝參數：

```bash
python get_install_parameters.py \
  --buffer-mem 2048 \
  --max-ram-scratch 2048 \
  --max-rk-rp-length 128 \
  --max-cred-blob-len 128 \
  --large-blob-store-size 2048 \
  --cache-pin-token \
  --protect-against-reset

```

**參數核心功能解析**

| 參數 | 設定值 | 功能解析 |
| --- | --- | --- |
| `--buffer-mem` | `2048` | **最核心關鍵**。將 RAM 緩衝區由預設 1024 調大至 2048，徹底解決微軟發送超大 WebAuthn 請求封包時引發的 `REQUEST_TOO_LARGE` 錯誤。 |
| `--max-ram-scratch` | `2048` | 將運算暫存區（Working Memory）調大至 2048 位元組。J3R180 具備充裕的 RAM（約 8KB+），調大可加速運算並大幅降低 Flash 寫入損耗。 |
| `--max-rk-rp-length` | `128` | 將網站識別碼（RP ID）最大長度由預設 32 擴增至 128 位元組，避免登入長網域（如 Azure / Entra ID 租用戶子網域）時截斷或溢位。 |
| `--max-cred-blob-len` | `128` | 將每個憑證附加的 Blob 資料上限由 32 提高至 128 位元組，相容更多擴充資料。 |
| `--large-blob-store-size` | `2048` | 將 Large Blob 空間由 1024 調大至 2048 位元組，提供充裕的延伸功能儲存空間。 |
| `--cache-pin-token` | - | 快取 PIN 授權權限，避免短時間內多次驗證時反覆彈出輸入視窗。 |
| `--protect-against-reset` | - | 啟用防誤重置機制，要求跨越兩次重新供電插拔才能重置，防止背景程式誤清空卡片。 |

---

### 第三階段：載入 CAP 套件（Load）

使用預設出廠金鑰（`404142434445464748494A4B4C4D4E4F`）將 `FIDO2.cap` 寫入卡片記憶體：

```bash
java -jar gp.jar \
  --key-enc 404142434445464748494A4B4C4D4E4F \
  --key-mac 404142434445464748494A4B4C4D4E4F \
  --key-dek 404142434445464748494A4B4C4D4E4F \
  --load FIDO2.cap

```

* **預期輸出**：顯示 `FIDO2.cap loaded: us.q3q.fido2 A000000647`

---

### 第四階段：建立 Applet 實體（Create）

帶入第二階段生成的 Hex 參數字串，並賦予 `CardReset` 權限建立實體：

```bash
java -jar gp.jar \
  --key-enc 404142434445464748494A4B4C4D4E4F \
  --key-mac 404142434445464748494A4B4C4D4E4F \
  --key-dek 404142434445464748494A4B4C4D4E4F \
  --applet A0000006472F0001 \
  --package A000000647 \
  --create A0000006472F0001 \
  --params <第二階段生成的HEX字串> \
  --privs CardReset

```

---

### 第五階段：驗證安裝狀態

查詢卡片內部應用程式清單：

```bash
java -jar gp.jar \
  --key-enc 404142434445464748494A4B4C4D4E4F \
  --key-mac 404142434445464748494A4B4C4D4E4F \
  --key-dek 404142434445464748494A4B4C4D4E4F \
  -list

```

**確認回傳清單包含以下項目：**

* `APP: A0000006472F0001 (SELECTABLE)`（下方具有 `Privs: CardReset`）
* `PKG: A000000647 (LOADED)`

---

### 第六階段：更換管理金鑰與鎖定生產模式（Key Rotation & SECURED Lock）

1. **替換為私有管理金鑰**
```bash
java -jar gp.jar \
  --key-enc 404142434445464748494A4B4C4D4E4F \
  --key-mac 404142434445464748494A4B4C4D4E4F \
  --key-dek 404142434445464748494A4B4C4D4E4F \
  --lock-enc <YOUR_NEW_ENC> \
  --lock-mac <YOUR_NEW_MAC> \
  --lock-dek <YOUR_NEW_DEK>

```


2. **切換至 SECURED 生產模式**
```bash
java -jar gp.jar \
  --key-enc <YOUR_NEW_ENC> \
  --key-mac <YOUR_NEW_MAC> \
  --key-dek <YOUR_NEW_DEK> \
  -s 80F0800F

```


3. **最終狀態確認**
```bash
java -jar gp.jar \
  --key-enc <YOUR_NEW_ENC> \
  --key-mac <YOUR_NEW_MAC> \
  --key-dek <YOUR_NEW_DEK> \
  -list

```


* **確認目標**：輸出第一行需顯示 `ISD: A000000151000000 (SECURED)`。



---

### 第七階段：重新插拔與 Passkey 初始化

1. **實體重新插拔（Cold Reset）**：將讀卡機連同卡片從 USB 埠拔除，靜置 3 秒後重新插入，強制卡片切換至標準 FIDO2 / CTAP2 模式。
2. **初始化 PIN 碼**：前往 **Windows 設定** → **帳戶** → **登入選項** → **安全性金鑰** → 點擊 **管理** 並設定 PIN 碼。
3. **帳號綁定測試**：前往 Google 帳戶安全性設定或 Microsoft 帳號管理頁面新增 Passkey 進行驗證。
