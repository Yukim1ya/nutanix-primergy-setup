# PRAID EP400i 論理ドライブ作成パラメータシート

**対象機器**: Fujitsu PRIMERGY RX2540 M2  
**RAIDコントローラー**: Fujitsu PRAID EP400i  
**用途**: Nutanix AOS / Community Edition インストール用  
**作成日**: 2026-05-31  

---

## 構成概要

Nutanix はハードウェア RAID を通じたディスク管理に対応していないため、  
物理ディスク1台につき1つの **RAID 0（シングルディスク構成）** を作成し、  
ディスクを論理ドライブとして OS（Nutanix）に見せる構成を採用する。

```
SSD① ─── RAID 0（1台構成）─── 論理ドライブ #0 ─── Nutanix
SSD② ─── RAID 0（1台構成）─── 論理ドライブ #1 ─── Nutanix
```

---

## パラメータシート

| # | パラメータ名 | 設定値 | 選択肢 |
|---|------------|--------|--------|
| 1 | Virtual Drive size | **ディスク全容量（デフォルト）** | 数値入力 |
| 2 | Virtual Drive size unit | **GB** | MB / GB / TB |
| 3 | Strip size | **64 KB** | 8KB / 16KB / 32KB / 64KB / 128KB / 256KB / 512KB / 1MB |
| 4 | Read policy | **No Read Ahead** | Read Ahead / No Read Ahead / Adaptive Read Ahead |
| 5 | Write policy | **Write Through** | Write Back / Write Through / Write Back with BBU |
| 6 | I/O policy | **Direct** | Cached / Direct |
| 7 | Access policy | **Read Write** | Read Write / Read Only / Blocked |
| 8 | Drive cache | **Enable** | Enable / Disable / No Change |
| 9 | Disable background initialization | **No（無効化しない）** | Yes / No |
| 10 | Default initialization | **Fast** | None / Fast / Full |
| 11 | Emulation type | **Disable** | Default / Disable / Force |

---

## 各パラメータ解説

### 1. Virtual Drive size（論理ドライブサイズ）

論理ドライブとして作成するサイズを指定する。

- **デフォルト（全容量）を推奨**
- 物理ディスクの全容量を1つの論理ドライブに割り当てる
- Nutanix が利用できる容量を最大化するために全容量を使用する
- 一部の容量だけを指定することで複数の論理ドライブに分割することも可能だが、今回は不要

---

### 2. Virtual Drive size unit（サイズ単位）

Virtual Drive size で指定した数値の単位。

- **GB を推奨**
- SSDのサイズに合わせて MB / GB / TB から選択する
- 通常は GB または TB を使用

---

### 3. Strip size（ストライプサイズ）

RAID 構成時にデータを分割して書き込む際の1チャンクのサイズ。

- **64KB を推奨**
- 今回は RAID 0（1台構成）のため、ストライピングの効果はないが設定値は必要
- Nutanix は独自のI/O管理（Stargate）を行うため、コントローラー側のストライプサイズの影響は小さい
- 64KB は業界標準の汎用的な値

| ストライプサイズ | 向いているワークロード |
|---------------|-------------------|
| 8KB〜32KB | 小さいランダムI/O（OLTP、データベース） |
| **64KB** | **汎用（今回の推奨）** |
| 128KB〜1MB | 大きいシーケンシャルI/O（動画、バックアップ） |

---

### 4. Read policy（読み取りポリシー）

コントローラーのキャッシュを使った先読み（Read Ahead）の動作を制御する。

- **No Read Ahead を推奨**
- Nutanix は Stargate が独自に読み取りキャッシュを管理するため、コントローラー側の先読みは不要かつ競合の原因になる

| 設定値 | 動作 |
|--------|------|
| **No Read Ahead** | 先読みなし。要求されたデータのみ読む |
| Read Ahead | 次に必要なデータを予測してキャッシュに先読み |
| Adaptive Read Ahead | アクセスパターンを学習して自動的に先読みを切り替え |

---

### 5. Write policy（書き込みポリシー）

書き込みデータをコントローラーキャッシュに保持するかどうかを制御する。

- **Write Through を推奨**（BBU なしの場合は必須）
- Write Back はコントローラーキャッシュに書き込んだ時点でホストに完了を通知するため高速だが、電源断時にキャッシュ内のデータが失われるリスクがある
- PRAID EP400i にバッテリーバックアップユニット（BBU）が搭載されていない場合は Write Through を選択すること

| 設定値 | 動作 | リスク |
|--------|------|--------|
| **Write Through** | ディスクへの書き込み完了後に通知。低速だが安全 | なし |
| Write Back | キャッシュへの書き込み完了後に通知。高速 | 電源断でデータ損失の可能性 |
| Write Back with BBU | BBU 搭載時のみ Write Back を使用。BBU 劣化時は自動で Write Through に切替 | BBU 必要 |

> **BBU の確認方法**: iRMC の ServerView RAID Manager でコントローラー情報を確認。  
> バッテリーの表示がある場合は Write Back with BBU も選択可能。

---

### 6. I/O policy（I/Oポリシー）

読み取りデータをコントローラーキャッシュに格納するかどうかを制御する。

- **Direct を推奨**
- Nutanix はコントローラーキャッシュを経由しない Direct I/O を推奨している
- Cached にするとコントローラーキャッシュに読み取りデータが格納されるが、Nutanix 自身のキャッシュと二重になり非効率

| 設定値 | 動作 |
|--------|------|
| **Direct** | コントローラーキャッシュを経由しない |
| Cached | 読み取りデータをコントローラーキャッシュに格納 |

---

### 7. Access policy（アクセスポリシー）

論理ドライブへのアクセス種別を制限する。

- **Read Write を推奨**
- Nutanix が読み書き両方を行う通常の設定

| 設定値 | 動作 |
|--------|------|
| **Read Write** | 読み書き両方を許可（通常運用） |
| Read Only | 読み取りのみ許可 |
| Blocked | アクセス完全禁止（保守用途） |

---

### 8. Drive cache（ドライブキャッシュ）

物理ディスク（SSD）自身が内蔵するキャッシュの有効/無効を制御する。

- **Enable を推奨**
- SSD 内蔵キャッシュを有効化することで SSD 本来の性能を発揮できる
- HDD では電源断時のリスクがあるが、SSD では一般的に有効化して問題ない

| 設定値 | 動作 |
|--------|------|
| **Enable** | ディスク内蔵キャッシュを有効化 |
| Disable | ディスク内蔵キャッシュを無効化 |
| No Change | ディスクの現在の設定を変更しない |

---

### 9. Disable background initialization（バックグラウンド初期化の無効化）

論理ドライブ作成後にバックグラウンドで行われる初期化処理を無効化するかどうかを設定する。

- **No（無効化しない＝初期化を実行する）を推奨**
- バックグラウンド初期化はディスクの一貫性チェックとパリティ情報の構築を行う
- RAID 0 では必須ではないが、実行しておくことでディスクの健全性を確認できる
- 「Disable background initialization = No」は「無効化しない」= 「初期化を実行する」の意味であることに注意

---

### 10. Default initialization（デフォルト初期化）

論理ドライブ作成時のデータ初期化方法を指定する。

- **Fast を推奨**
- SSD は物理的なゼロ埋めが不要なため Fast で十分
- Full は全セクターに書き込みを行うため時間がかかる（HDD での旧データ消去に使用）

| 設定値 | 動作 | 所要時間 |
|--------|------|---------|
| None | 初期化なし | 即時 |
| **Fast** | メタデータのみ初期化（論理的にゼロ埋め） | 数秒〜数分 |
| Full | 全セクターに物理的に書き込み | 数時間（容量による） |

---

### 11. Emulation type（エミュレーションタイプ）

論理ドライブのセクターサイズエミュレーションを指定する。

- **None を推奨**
- 現代の OS・ハイパーバイザーは 512e / 4Kn 等を直接扱えるため、エミュレーションは不要
- Nutanix AHV は標準的なセクターサイズに対応しているため None で問題ない

| 設定値 | 動作 |
|--------|------|
| Default | コントローラーのデフォルト設定に従う（環境依存） |
| **Disable** | エミュレーション無効。ディスク本来のセクターサイズをそのまま OS に提示（推奨） |
| Force | エミュレーションを強制有効化。古い OS との互換性が必要な場合に使用 |

---

## 作業手順

### 論理ドライブ作成（SSD 1台につき同じ手順を繰り返す）

1. BIOS/UEFI セットアップ → `Create Virtual Disk` を選択
2. RAID レベルとして `RAID 0` を選択
3. 物理ディスクを **1台だけ** 選択
4. 上記パラメータシートの設定値を入力
5. `Create` または `Apply` で作成
6. SSD の枚数分（今回は2台）繰り返す

### 作成後の確認

- 論理ドライブが2つ作成されていること
- 各論理ドライブの状態が `Optimal` または `Online` になっていること

---

## Nutanix インストール時の注意

論理ドライブ作成後、Nutanix インストーラーがディスクを **HDD として誤認識する場合**がある。  
その場合はインストーラーを `Ctrl+C` で中断し、以下を実行してから再開する。

```bash
# SSD として認識させる（リブート不要）
echo 0 > /sys/block/sda/queue/rotational
echo 0 > /sys/block/sdb/queue/rotational

# 確認（0 が返れば SSD 認識）
cat /sys/block/sda/queue/rotational
```

---

## 環境情報

| 項目 | 内容 |
|------|------|
| サーバー | Fujitsu PRIMERGY RX2540 M2 |
| RAIDコントローラー | PRAID EP400i |
| コントローラーチップ | PCI Vendor/Device: 0x1000 / 0x005D |
| ファームウェア版数 | 4.660.00-8175 |
| 物理ディスク | SATA SSD × 2 |
| インストールOS | Nutanix AOS / Community Edition |
| 構成 | RAID 0 × 2（シングルディスク構成） |
