# Nutanix CE ブータブルUSB作成 作業記録

**対象機器**: Fujitsu PRIMERGY RX2540 M2  
**インストール対象**: Nutanix AOS / Community Edition  
**ISOファイル**: `phoenix.x86_64-fnd_5.6.1_patch-aos_6.8.1_ga.iso`（約7GB）  
**作成日**: 2026-05-31

---

## 背景・目的

Nutanix CE のインストールには **Phoenix ISO** と呼ばれる Foundation インストーラーを使用する。  
このISOはブータブルUSBから起動し、サーバーにNutanixをインストールする。

### Phoenix ISO の特性（重要）

Phoenix ISO は**ハイブリッドISO**ではなく、起動後にインストーラー自身が  
**USBデバイス上のISOファイルを検索して読み込む**という動作をする。

このため、一般的なOS用ブータブルUSB作成ツールが採用する「ISOを解凍してFAT32に展開する」  
方式では動作しない。**ISOファイルをそのままバイト列でUSBに書き込む（rawイメージ書き込み）**  
が必要となる。

---

## 試行記録

### 試行1：Rufus（ISOモード）

**目的**: 最も一般的なブータブルUSB作成ツールでUSBを作成する  
**結果**: ❌ 失敗  
**エラー**: サーバー起動時に `Failed to find Phoenix ISO` と表示される  
**原因**: ISOモードではISOが解凍されてFAT32パーティションに展開される。  
　　　　ISOファイル自体がUSB上に存在しないため、Phoenixインストーラーが自分自身を見つけられない。

---

### 試行2：Rufus（DDモード）

**目的**: Rufusには「DDイメージモード」があり、rawイメージ書き込みが可能  
**結果**: ❌ 失敗  
**エラー**: DDモード選択ダイアログが表示されない  
**原因**: Rufusが Phoenix ISOをハイブリッドISOとして認識しないため、  
　　　　DDモードの選択肢が表示されない。  
　　　　`Alt+クリック`や `.imgにリネーム` などの回避策を試みたが効果なし。

---

### 試行3：Rufus 3.22（旧バージョン）

**目的**: 旧バージョンのRufusでは ISOを選択時に「ISOモード / DDモード」を明示的に選択するダイアログが表示される  
**結果**: ❌ 失敗  
**エラー**: 3.22でもDDモードのダイアログが表示されない  
**原因**: Phoenix ISOの構造がRufusのハイブリッドISO判定基準を満たさないため。

---

### 試行4：Balena Etcher

**目的**: ISOをそのままデバイスに書き込むことに特化したツールを使用する  
**結果**: ❌ 失敗  
**エラー**: `error starting flasher sidecar process`  
**原因**: ウイルス対策ソフトまたはアクセス権限の問題でサイドカープロセスが起動できない。  
　　　　`Only allocated partitions` のOFF/ONを変えて試みたが解消されなかった。

---

### 試行5：Win32DiskImager

**目的**: シンプルなrawイメージ書き込みツールを使用する  
**結果**: ❌ 失敗  
**原因**: 詳細不明。書き込み完了後もブートメディアとして認識されなかった。

---

### 試行6：Ventoy（normal mode）

**目的**: VentoyはUSBにISOファイルをコピーするだけで多数のISOを起動できるツール。  
　　　　ISOモード/DDモードの概念がなく、コピーするだけでよい  
**手順**:
1. Ventoy2Disk.exe でUSBをVentoy化（ExFATパーティションが作成される）
2. `phoenix.iso` をExFATパーティションにコピー
3. USBからブート → Ventoyメニューで `Boot in normal mode` を選択

**結果**: ❌ 失敗  
**エラー**: `Failed to find Phoenix ISO`  
**原因**: Ventoy normal modeではISOをループマウントして起動するため、  
　　　　Phoenixインストーラーからみると自分自身のISOファイルが見えない状態になる。

---

### 試行7：Ventoy（grub2 mode）

**目的**: grub2モードでは起動の仕組みが異なるため、normal modeの問題を回避できる可能性がある  
**結果**: ❌ 失敗  
**エラー**:
```
/ventoy/hook/rhel7/ventoy-hook.sh: line 87: ls: not found
grep: /etc/os-release: No such file or directory
sed: /lib/dracut-lib.sh: No such file or directory
cp: can't create '/lib/dracut/hooks/...': No such file or directory
Failed to init, dropping to a shell
can't access tty; job control turned off
```
**原因**: Phoenix ISOは独自のinitramfsを使用しており、Ventoyのhookスクリプトが  
　　　　前提とする標準的なLinux環境（`/etc/os-release`、`dracut-lib.sh` 等）が存在しない。  
　　　　**Ventoyは Phoenix ISOと互換性がない。**

---

### 試行8：PowerShell FileStream rawイメージ書き込み ← 現在実施中

**目的**: Windowsに標準搭載のPowerShellを使って `dd` コマンド相当のrawイメージ書き込みを行う  
**手順**:

1. diskpartでUSBのパーティションを削除（Windowsのロックを解除するため）
   ```cmd
   diskpart
   select disk 4
   clean
   exit
   ```

2. PowerShellでISOをPhysicalDriveに直接書き込む
   ```powershell
   $iso = [System.IO.FileStream]::new("C:\path\to\phoenix.iso", [System.IO.FileMode]::Open)
   $disk = [System.IO.FileStream]::new("\\.\PhysicalDrive4", [System.IO.FileMode]::Open, [System.IO.FileAccess]::ReadWrite)
   $buf = New-Object byte[](4MB)
   while(($n=$iso.Read($buf,0,$buf.Length)) -gt 0){ $disk.Write($buf,0,$n) }
   $iso.Close(); $disk.Close()
   Write-Host "完了"
   ```

**備考**:
- `Set-Disk -IsOffline $true` はリムーバブルメディアには使用不可（エラーになる）
- `diskpart clean` でパーティションテーブルを削除することでWindowsのロックが解除される
- 約7GBのため書き込みに20〜30分かかる
- 実行中は何も表示されない。`完了` が表示されれば成功

**結果**: ✅ 成功（書き込み完了）

**補足**: 書き込み完了後、Windows PC からUSBが認識されなくなるが**これは正常**。  
rawイメージはWindowsのファイルシステム（FAT32/NTFS）ではないため、エクスプローラーから見えない。  
「フォーマットしますか？」と表示される場合もあるが無視してよい。  
そのままサーバーのUSBポートに挿してBIOSからUSBブートすれば Phoenix が起動する。

---

## 次のステップ

1. USBをPRIMERGYサーバーに挿す
2. BIOS起動メニューでUSBを選択してブート
3. Phoenix インストーラーが起動することを確認
4. Nutanix CE のインストール開始

---

## 技術メモ

### なぜ多くのツールが失敗するのか

| ツール | 書き込み方式 | 結果 |
|--------|------------|---------|
| Rufus（ISOモード） | ISO解凍 → FAT32展開 | ❌ ISOファイルが存在しない |
| Rufus（DDモード） | rawイメージ書き込み | ❌ Phoenix ISOをハイブリッドISOと認識しない |
| Balena Etcher | rawイメージ書き込み | ❌ sidecarプロセス起動失敗（AV/権限問題） |
| Win32DiskImager | rawイメージ書き込み | ❌ 詳細不明 |
| Ventoy normal | ISOループマウント | ❌ Phoenixが自分自身を見つけられない |
| Ventoy grub2 | ISO grub2起動 | ❌ Phoenix独自initramfsと非互換 |
| **PowerShell FileStream** | **rawイメージ書き込み** | **✅ 成功** |

### Nutanix SSD誤認識の回避

インストーラーがSSDをHDDと誤認識した場合は、`Ctrl+C` で中断後に以下を実行：

```bash
echo 0 > /sys/block/sda/queue/rotational
echo 0 > /sys/block/sdb/queue/rotational
```

詳細は `praid-ep400i-parameter-sheet.md` を参照。
