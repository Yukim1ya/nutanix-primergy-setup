# Nutanix CE インストール失敗 調査レポート

**対象機器**: Fujitsu PRIMERGY RX2540 M1  
**ISOファイル**: `phoenix.x86_64-fnd_5.6.1_patch-aos_6.8.1_ga.iso`  
**エラー**: `Failed to find Phoenix ISO`  
**発生日**: 2026-06-06

---

## エラーログ全文（抜粋・整形）

```
10.747890  usb-storage: registered new interface driver usb-storage
10.757606  i8042: PNP: No PS/2 controller found.
10.878127  rtc_cmos 00:00: setting system clock to 2026-06-06T17:54:30 UTC
11.012663  usb 1-1: new high-speed USB device number 2 using ehci-pci
11.303381  hub 1-1:1.0: USB hub found
11.302650  hub 1-1:1.0: 6 ports detected
11.316565  Run /init as init process

2026-06-06-17:54:31-UTC  kernel.core_pattern = /tmp/core-...
2026-06-06-17:54:31-UTC  Loading drivers
2026-06-06-17:54:33-UTC  Looking for extra/140e.ko          ← 注目点①
2026-06-06-17:54:33-UTC  Getting DHCP address for phoenix
2026-06-06-17:54:33-UTC  Waiting for DHCP lease
2026-06-06-17:54:37-UTC  Looking for device containing      ← 4秒でDHCP打ち切り
115/151 Waiting for Phoenix ISO to be available ... Failed to find Phoenix ISO.
2026-06-06-17:55:11-UTC  livecd files not found.
2026-06-06-17:55:11-UTC  Community Edition has encountered an error during installation.
```

---

## ログ詳細分析

### ① CPUマイクロコードからハードウェア世代を特定

ログ内のマイクロコードシグネチャ:

```
microcode: sig=0x306f2, Pf=0x1, revision=0x44
```

| フィールド | 値 | 解釈 |
|-----------|-----|------|
| sig | `0x306f2` | Family 6, Model `0x3f`, Stepping 2 |
| Model `0x3f` | Haswell-EP | Intel Xeon E5-2600 v3 世代 |

**PRIMERGY RX2540 M1 = Haswell-EP（Xeon E5-2600 v3）世代で確定**。  
M2 は Broadwell-EP（Xeon E5-2600 v4, sig=`0x406f1`）のため、USB作成時の想定機種（M2）と実際の機種（M1）が異なります。

---

### ② `extra/140e.ko` が見つからない

```
2026-06-06-17:54:33-UTC  Looking for extra/140e.ko
```

Phoenix initramfs は、標準カーネルに含まれないドライバを **ブートUSB内の `extra/` ディレクトリ** から追加ロードする仕組みを持っています。  
`140e.ko` は PCI デバイスID `0x140e` に対応するドライバと推定されます。

**デバイスIDの候補（PRIMERGY RX2540 M1 搭載コンポーネント）:**

| 可能性 | PCI ID | 影響 |
|--------|--------|------|
| NIC（Broadcom BCM5720等、モデルによる） | 要実機確認 | ドライバなし → ネットワーク不可 → DHCP失敗 |
| SASコントローラ（Fujitsu D3116/D3216等） | 要実機確認 | ドライバなし → 内蔵HDD/SSD不可 → インストール先なし |
| RAID コントローラ（LSI MegaRAID等） | 要実機確認 | 同上 |

> `140e.ko` が見つからなくても Phoenix は処理を続行しますが、対応デバイスが使用不能になります。

---

### ③ DHCP 待機時間が 4 秒で打ち切られている

```
17:54:33  Getting DHCP address for phoenix
17:54:33  Waiting for DHCP lease
17:54:37  Looking for device containing ...   ← わずか 4 秒後
```

NICドライバが見つからない場合、ネットワークインターフェースが初期化されないため DHCP 取得は必然的に失敗します。  
DHCP が取得できれば Phoenix はネットワーク経由（HTTP）でも ISO を検索しますが、この経路も使えていません。

---

### ④ USBがUSB 2.0（EHCI）で検出されている

```
usb 1-1: new high-speed USB device number 2 using ehci-pci
```

- `ehci-pci` = USB 2.0 コントローラ（EHCI）
- `high-speed` = USB 2.0（480 Mbps）

USB は検出されており、ブート自体は成功しています。ただし USB 3.0 コントローラ（xHCI）が使われていない点は注目です。  
Phoenix の initramfs に xHCI ドライバが含まれない場合、USB 3.0 ポートに挿したデバイスが EHCI 経由（USB 2.0 互換モード）で動作します。USB自体は読み取れているため、直接の原因ではありません。

---

## 根本原因の分析

### 原因1（最有力）：Phoenix ISO 検索メカニズムの誤解

**Phoenix のブートプロセス（2フェーズ構造）:**

```
フェーズ1（完了）: USBのブートセクタから最小カーネル + initramfs を起動
                  ↓ この時点では起動成功
フェーズ2（失敗）: initramfs が "Phoenix ISOファイル" を全ストレージデバイスから検索
                  ↓ ISOファイルが見つからず失敗
```

**raw write の問題点:**

PowerShell FileStream による raw write を行った場合、USB の構造は以下のようになります:

```
USB全体 = Phoenix ISO イメージ（バイト列そのまま）
           → ISO9660 ファイルシステム（ブート可能）
           → Phoenix initramfs・カーネルが含まれる
           → "phoenix.iso" というファイルは含まれない
```

Phoenix initramfs がブート後に探すのは **`phoenix.x86_64-*.iso` というファイル名のISOファイル** です。  
raw write されたUSBには ISO9660 ファイルシステムは存在しますが、その中に「ISOファイル」は存在しません。  
**ISOの中にISOを内包することはできないため、raw writeだけでは自己参照になりフェーズ2が成立しない**のが根本原因です。

### 原因2（補助的）：`140e.ko` ドライバ欠如

NIC ドライバが不足している場合、ネットワーク経由の ISO 取得も不可能になります。  
ローカルストレージ検索とネットワーク検索の両方が失敗し、エラーが確定します。

### 原因3（背景）：M1 vs M2 のハードウェア差異

| 項目 | M1（実際） | M2（USB作成時の想定） |
|------|-----------|------------------|
| CPU世代 | Haswell-EP (v3) | Broadwell-EP (v4) |
| チップセット | Intel C610 | Intel C610 |
| リリース年 | 2015年 | 2016年 |
| Nutanix CE HCL | **未確認** | 検証済みの実績あり |
| NIC（標準） | Broadcom/Intel（構成依存） | 同左 |

Nutanix CE のハードウェア互換性リスト（HCL）に M1 が含まれているか未確認です。  
M1 特有のデバイス（NIC・ストレージコントローラ）に対するドライバが Phoenix 5.6.1 に含まれていない可能性があります。

---

## 対処方法（優先順位順）

### 対処A：2本目のUSBにISOファイルを配置する【最優先・最も簡単】

Phoenix initramfs は全ストレージデバイスを検索するため、  
**FAT32フォーマットの2本目のUSBにISOファイルをコピーして同時挿入**することで解決できます。

**手順:**

1. 空のUSBメモリを用意（8GB以上推奨）
2. FAT32 でフォーマット
3. `phoenix.x86_64-fnd_5.6.1_patch-aos_6.8.1_ga.iso` をそのままコピー（解凍不要）
4. ブートUSB（Trial 8 で作成済み）とISOファイル用USBの **2本を同時に挿入**
5. ブートUSBから起動

```
構成イメージ:
  USB-A（ブート用）: raw write済み → フェーズ1担当
  USB-B（ISO用）  : FAT32 + ISOファイル → フェーズ2でPhoenixが自動検出
```

> **注意**: FAT32 は最大4GBのファイルサイズ制限があります。ISOが7GBの場合は **exFAT** でフォーマットしてください。

```powershell
# PowerShell でexFATフォーマット（管理者権限）
# DiskpartでUSBを確認後
Format-Volume -DriveLetter X -FileSystem exFAT -NewFileSystemLabel "PHOENIX_ISO" -Confirm:$false
```

---

### 対処B：PRIMERGY RX2540 M1 の HCL 確認

Nutanix CE の公式互換ハードウェアリストを確認します。

- [Nutanix CE Hardware Compatibility List](https://www.nutanix.com/products/community-edition/hardware-compatibility-list)
- RX2540 **M1** が掲載されているか確認（M2 と M1 は別エントリ）
- 掲載がない場合、CE インストール自体が非サポートのため注意

---

### 対処C：搭載NIC/ストレージコントローラのPCI IDを確認する

BIOS/UEFI でデバイス情報を確認するか、別OS（Linux LiveUSB等）から確認します。

```bash
# Linux LiveUSB から確認
lspci -nn | grep -E "(Ethernet|RAID|SAS|SATA)"
```

出力例の読み方:
```
02:00.0 Ethernet controller [0200]: Broadcom Limited BCM5720 [14e4:165f] (rev 01)
                                                                ^^^^^^^^
                                                          ベンダID:デバイスID
```

`140e` がどのデバイスかを特定し、Nutanix CE が対応しているか確認します。

---

### 対処D：extra/ ディレクトリにドライバを配置する

対処CでPCI IDが特定できた場合、Phoenix の `extra/` ディレクトリに追加ドライバを配置します。

1. Linux LiveUSB から対象カーネルバージョンのドライバモジュール（`.ko`）を取得
2. ブートUSB（ISO9660なので書き込み不可）ではなく、**FAT32 USB に `extra/` ディレクトリを作成して配置**
3. 対処Aと組み合わせて実施

```
USB-B（FAT32）の構成:
  /extra/140e.ko       ← 追加ドライバ
  /phoenix.x86_64-fnd_5.6.1_patch-aos_6.8.1_ga.iso  ← ISOファイル
```

---

### 対処E：Phoenix をネットワーク経由で提供する【高難度・最終手段】

DHCP サーバー + HTTP サーバーを立て、Phoenix が ISO をダウンロードできるようにします。  
NIC ドライバの問題が解決していれば有効ですが、インフラ構築が必要なため難易度が高いです。

---

## 推奨アクション

| 優先度 | アクション | 難易度 | 期待効果 |
|--------|-----------|--------|---------|
| ① | **対処A**: FAT32 USB に ISO コピー（2本差し） | 低 | 原因1を直接解消 |
| ② | **対処B**: M1 HCL 確認 | 低 | サポート対象か確認 |
| ③ | **対処C**: `lspci -nn` でPCI ID特定 | 中 | `140e.ko` の正体を特定 |
| ④ | **対処D**: extra/ にドライバ配置 | 高 | 原因2を解消 |
| ⑤ | **対処E**: PXEブート環境構築 | 高 | 全経路の代替手段 |

**まず対処A（2本差し）を試してください。** 多くの場合これで解決します。

---

## 参考

| 項目 | 内容 |
|------|------|
| 使用ISO | phoenix.x86_64-fnd_5.6.1_patch-aos_6.8.1_ga.iso |
| USB作成方法 | PowerShell FileStream raw write（Trial 8） |
| 詳細: USB作成記録 | [bootable-usb-creation.md](bootable-usb-creation.md) |
| Nutanix CE HCL | https://www.nutanix.com/products/community-edition/hardware-compatibility-list |
