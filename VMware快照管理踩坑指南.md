# VMware 快照管理踩坑指南 — 快照膨胀、删除卡住的处理

> 快照是 VMware 运维中最常用也最容易翻车的功能。本文从原理出发，系统梳理快照膨胀、删除卡住、合并失败等常见问题，附带完整的排查流程和处理方案。

**快照不是备份！快照不是备份！快照不是备份！** 快照只是一个时间冻结点，不能替代 Veeam / NBU 等专业备份工具。长期保留快照 = 给自己埋雷。

---

## 一、快照的工作原理

### 1.1 快照到底是什么

创建快照时，VMware 做了三件事：

1. **创建 delta 磁盘文件**（`-000001.vmdk`）— 原始磁盘变为只读，所有新的写操作都写入 delta 文件
2. **保存内存状态**（可选）— 生成 `.vmem` 和 `.vmsn` 文件
3. **记录快照树**（`vmname.vmsd`）— 维护快照的层级关系

delta 文件**只增不减**，哪怕 VM 里删了文件，delta 也不会缩小。

```
原始磁盘 (base.vmdk)  ← 只读
    └── delta-000001.vmdk  ← 第 1 个快照后的写入
        └── delta-000002.vmdk  ← 第 2 个快照后的写入
            └── delta-000003.vmdk  ← 当前写入点
```

### 1.2 关键文件说明

| 文件 | 说明 |
|------|------|
| `vmname.vmdk` | 原始磁盘描述符文件 |
| `vmname-flat.vmdk` | 原始磁盘数据文件（实际占用空间） |
| `vmname-000001.vmdk` | 第 1 层 delta 磁盘描述符 |
| `vmname-000001-delta.vmdk` | 第 1 层 delta 磁盘数据 |
| `vmname.vmsd` | 快照元数据（快照树结构） |
| `vmname-Snapshot1.vmsn` | 快照内存状态 |
| `vmname-Snapshot1.vmem` | 快照内存数据 |

### 1.3 删除快照时发生了什么

删除快照 = **合并（consolidate）** delta 文件到父磁盘：

- **删除中间快照**：delta 数据向下合并到子快照
- **删除当前快照（最新的）**：delta 数据向上合并到父磁盘
- **删除所有快照 / Revert**：所有 delta 依次合并到 base 磁盘

**这就是所有问题的根源** — 合并过程需要大量 I/O 和临时空间，时间与 delta 文件大小成正比。

---

## 二、快照膨胀（delta 文件暴涨）

### 2.1 为什么会膨胀

快照创建后，**所有对磁盘的写操作都写入 delta 文件**。delta 文件只增不减，即使 VM 内部删除了文件，delta 文件也不会缩小。

**膨胀速度取决于：**

| 因素 | 影响 |
|------|------|
| **VM 写入量** | 数据库、日志密集型 VM 膨胀最快 |
| **存活时间** | 快照放 3 天和 3 个月，天差地别 |
| **快照层数** | 多层叠加，每层独立膨胀 |
| **磁盘类型** | Thin Provisioning 的 VM 膨胀更不容易被发现 |

### 2.2 真实翻车案例

给一台 SQL Server VM 打了快照，"临时"用一下。**3 个月后**：

```
base.vmdk         = 200GB（只读，没变）
delta-000001.vmdk = 180GB（3 个月的写入全在这里）

数据存储使用率从 60% 飙到 95%，触发告警。
删除预计 8+ 小时。
```

### 2.3 膨胀的 5 大后果

1. **存储爆满** — delta 把存储写满，同一 datastore 上所有 VM 同时暂停（ALL paths down）
2. **删除巨慢** — 180GB 的 delta 合并可能要几小时到十几小时
3. **性能骤降** — 合并期间 I/O 被吃掉，VM 明显卡顿
4. **合并失败** — 存储空间不足时合并会失败，可能导致数据损坏
5. **vMotion 失败** — 带大快照的 VM 迁移会失败或极慢

### 2.4 预防措施

#### 规则 1：快照存活时间不超过 72 小时

这是 VMware 官方建议。超过 72 小时的快照应该被视为异常。

#### 规则 2：快照层数不超过 3 层

每增加一层快照，性能下降约 5-10%。超过 32 层会导致 VM 无法启动。

#### 规则 3：不要在数据库 VM 上长期保留快照

SQL Server、Oracle、MySQL 等写入密集型 VM 的 delta 膨胀速度极快。

#### 规则 4：监控快照状态

```powershell
# PowerCLI — 查找所有超过 24 小时的快照
Get-VM | Get-Snapshot | Where-Object {
    $_.Created -lt (Get-Date).AddHours(-24)
} | Select-Object VM, Name, Created, SizeGB |
  Sort-Object SizeGB -Descending |
  Format-Table -AutoSize
```

#### 规则 5：监控数据存储可用空间

数据存储可用空间低于 20% 时必须告警。低于 10% 时禁止任何快照操作。

---

## 三、快照删除卡住（Consolidate 失败）

### 3.1 典型表现

- vSphere Client 中"删除快照"任务进度条卡在 0% / 95% 不动
- 任务显示"Virtual machine consolidation is needed"
- VM 的快照管理器显示"无快照"，但 datastore 中仍有 delta 文件
- 删除快照后 VM 变得极慢甚至无响应

### 3.2 卡住的常见原因

| 原因 | 说明 |
|------|------|
| **存储空间不足** | 合并需要临时空间（约等于 delta 大小），空间不够就卡住 |
| **SCSI 锁冲突** | 其他主机持有磁盘锁，合并无法获取独占访问 |
| **CBT 损坏** | Changed Block Tracking 文件损坏导致合并异常 |
| **快照链断裂** | .vmsd 中的快照树与实际 delta 文件不一致 |
| **I/O 过载** | 存储 I/O 延迟过高（>30ms），合并操作超时 |
| **VM 内部快照感知** | VSS 相关问题（Windows VM 的 quiesce 快照） |

### 3.3 排查流程

```
快照删除卡住
    │
    ├─ Step 1: 检查存储空间
    │    └─ 可用空间 < delta 大小？→ 先腾空间
    │
    ├─ Step 2: 检查任务状态
    │    └─ vCenter Tasks 中是否有"consolidate"任务？
    │    └─ 有没有错误信息？
    │
    ├─ Step 3: 检查磁盘锁
    │    └─ vmkfstools -D 检查是否有锁冲突
    │
    ├─ Step 4: 检查 delta 文件一致性
    │    └─ ls -la *.vmdk 对比 .vmsd 中的记录
    │
    ├─ Step 5: 尝试 Consolidate
    │    └─ VM → Snapshots → Consolidate
    │
    └─ Step 6: 以上都不行？→ 手动处理（见下文）
```

---

## 四、6 大故障场景处理方案

### 4.1 场景一：存储空间不足导致合并卡住

**症状**：快照删除任务卡在 0%，datastore 使用率 > 95%

**处理**：腾空间 → 确保可用空间 > 最大 delta → 重试删除

```bash
# 1. SSH 登录 ESXi 主机
ssh root@esxi-host

# 2. 检查数据存储可用空间
df -h /vmfs/volumes/datastore1/

# 3. 找到最大的 delta 文件
find /vmfs/volumes/datastore1/ -name "*delta.vmdk" -exec ls -lh {} \; | sort -k5 -hr | head -20

# 4. 紧急腾空间：
#    方案 A：删除其他 VM 的过期快照
#    方案 B：Storage vMotion 迁移小 VM 到别的存储
#    方案 C：删除 VM 内部的无用大文件（如日志）
#    方案 D：临时扩容数据存储（如果是 NFS/iSCSI）

# 5. 确保可用空间 > delta 文件大小后，重试删除快照
```

**PowerCLI 紧急腾空间**：

```powershell
# 找出数据存储上所有快照，按大小排序
$ds = Get-Datastore "datastore1"
Get-VM -Datastore $ds | Get-Snapshot |
  Select-Object VM, Name, SizeGB, Created |
  Sort-Object SizeGB -Descending |
  Format-Table -AutoSize

# Storage vMotion 迁移小 VM 到其他存储
$vm = Get-VM "small-vm-01"
Move-VM -VM $vm -Datastore "datastore2" -Confirm:$false
```

### 4.2 场景二：快照删除卡在 95%

**症状**：进度条到 95% 不动，已经几个小时了

**原因**：95% 通常意味着数据合并已完成，卡在最后的元数据更新阶段（重命名 delta → base）

**处理步骤**：

```
1. 不要取消任务！取消可能导致数据损坏
2. 检查 ESXi 的 vmkernel.log：
   grep "snapshot" /var/log/vmkernel.log | tail -20
3. 检查 hostd.log：
   grep -i "consolidat" /var/log/hostd.log | tail -20
4. 如果日志显示"File is locked"→ 转 4.3 锁冲突处理
5. 如果日志无报错，只是慢 → 耐心等待，不要干预
6. 如果超过 24 小时仍无进展 → 重启 hostd 服务：
   /etc/init.d/hostd restart
   （注意：这会断开 vCenter 连接，但不影响 VM 运行）
```

### 4.3 场景三：SCSI 磁盘锁冲突

**症状**：日志中出现"Failed to open disk"或"File is locked"

**排查步骤**：

```bash
# 1. 找到 delta 文件路径
ls -la /vmfs/volumes/datastore1/vm-name/*.vmdk

# 2. 检查磁盘锁的持有者
vmkfstools -D /vmfs/volumes/datastore1/vm-name/vm-name-000001-delta.vmdk

# 输出示例：
# Lock [type 10c00001 offset 45580288 v 1850, hb offset 3866624
#   gen 3423, mode 1, owner 5e8b3c2a-xxx-xxx-xxx MAC_ADDRESS]
#                                                 ^^^^^^^^^^^^
#                                                 这是持有锁的主机 MAC

# 3. 通过 MAC 地址找到持有锁的 ESXi 主机
esxcli network nic list  # 对比各主机的 MAC

# 4. 释放锁的方法：
#    方案 A: 在持有锁的主机上重启 VM 的 vmx 进程
#    方案 B: vMotion 到其他主机后重试
#    方案 C: 如果是残留锁（没有进程使用），等 ~3 分钟自动释放
```

### 4.4 场景四：快照管理器显示"无快照"但 delta 文件仍存在

**症状**：快照树为空，但 VM 目录下仍有 `-000001.vmdk` 文件

**这是最危险的情况之一** — 快照元数据与实际文件不一致。

**排查步骤**：

```bash
# 1. SSH 到 ESXi，检查 VM 目录
cd /vmfs/volumes/datastore1/vm-name/
ls -la *.vmdk

# 2. 查看当前 VM 使用的磁盘链
grep -i "parentFileNameHint\|fileName" *.vmx *.vmdk

# 3. 查看 .vmsd 文件（快照元数据）
cat vm-name.vmsd
# 如果内容为空或只有 .encoding，说明元数据已被清除但 delta 未合并

# 4. 检查 VM 实际指向哪个 vmdk
grep "scsi" vm-name.vmx | grep vmdk
# 如果指向 vm-name-000001.vmdk，说明 VM 还在用 delta 文件

# 5. 在 vSphere Client 中尝试 Consolidate：
#    VM → Snapshots → Consolidate
```

**如果 Consolidate 也失败**：

```bash
# 手动合并 — 仅在有完整备份后操作！

# 1. 关闭 VM（必须关机，不能在线操作）
# 2. 克隆 delta 到新磁盘
vmkfstools -i vm-name-000001.vmdk vm-name-consolidated.vmdk -d thin

# 3. 编辑 .vmx，将磁盘指向新文件
#    scsi0:0.fileName = "vm-name-consolidated.vmdk"

# 4. 启动 VM，验证数据完整性

# 5. 确认无误后，删除旧的 delta 和 base 文件
```

### 4.5 场景五：CBT 损坏导致合并失败

**症状**：日志中出现"Cannot open the disk '/vmfs/…' or one of the snapshot disks"

**处理步骤**：

```bash
# 1. 关闭 VM
# 2. 禁用 CBT — 在 VM 高级设置中：
#   ctkEnabled = "false"
#   scsi0:0.ctkEnabled = "false"
```

```powershell
# 或用 PowerCLI 禁用 CBT
$vm = Get-VM "vm-name"
$spec = New-Object VMware.Vim.VirtualMachineConfigSpec
$spec.changeTrackingEnabled = $false
$vm.ExtensionData.ReconfigVM($spec)

# 删除 CTK 文件（SSH 到 ESXi）：
# rm /vmfs/volumes/datastore1/vm-name/*.ctk
```

```bash
# 3. 启动 VM
# 4. 重试删除快照
# 5. 如果需要，重新启用 CBT（备份软件需要）
```

### 4.6 场景六：VSS 快照（quiesce）失败

**症状**：Windows VM 创建 quiesce 快照时失败，或快照创建成功但删除卡住

**原因**：VMware Tools 中的 VSS Provider 与 Windows VSS 服务冲突

**处理步骤**：

```powershell
# 在 Windows VM 内部执行：

# 1. 检查 VSS Provider 状态
vssadmin list providers

# 2. 检查 VSS Writers 状态
vssadmin list writers
# 找 State 不是 "Stable" 的 Writer

# 3. 重启 VMware VSS 相关服务
Restart-Service "VMware Snapshot Provider" -Force
Restart-Service "VSS" -Force

# 4. 清理残留的 VSS Shadow Copies
vssadmin delete shadows /all /quiet

# 5. 如果问题持续，禁用 quiesce 快照改用普通快照
# 在创建快照时取消勾选"Quiesce guest file system"
```

---

## 五、PowerCLI 快照管理脚本

### 5.1 快照健康检查

```powershell
# ============================================================
# VMware 快照健康检查脚本
# 用法：连接 vCenter 后运行
# ============================================================

# Connect-VIServer -Server vcenter.company.com

$WarningHours = 24
$CriticalHours = 72
$MaxLayers = 3
$Now = Get-Date

Write-Host "`n=== VMware 快照健康检查 ===" -ForegroundColor Cyan
Write-Host "时间: $($Now.ToString('yyyy-MM-dd HH:mm:ss'))`n"

# 获取所有快照
$allSnapshots = Get-VM | Get-Snapshot

if ($allSnapshots.Count -eq 0) {
    Write-Host "[OK] 未发现任何快照" -ForegroundColor Green
    return
}

Write-Host "发现 $($allSnapshots.Count) 个快照:`n" -ForegroundColor Yellow

$results = foreach ($snap in $allSnapshots) {
    $ageHours = ($Now - $snap.Created).TotalHours
    $level = if ($ageHours -gt $CriticalHours) { "CRITICAL" }
             elseif ($ageHours -gt $WarningHours) { "WARNING" }
             else { "OK" }

    [PSCustomObject]@{
        VM       = $snap.VM.Name
        Snapshot = $snap.Name
        Created  = $snap.Created.ToString('yyyy-MM-dd HH:mm')
        AgeHours = [math]::Round($ageHours, 1)
        SizeGB   = [math]::Round($snap.SizeGB, 2)
        Level    = $level
    }
}

# 按风险等级和大小排序输出
$results | Sort-Object @{e='Level';Descending=$true}, @{e='SizeGB';Descending=$true} |
    Format-Table -AutoSize

# 统计
$criticals = ($results | Where-Object Level -eq "CRITICAL").Count
$warnings = ($results | Where-Object Level -eq "WARNING").Count
$totalSizeGB = ($results | Measure-Object SizeGB -Sum).Sum

Write-Host "统计: 严重=$criticals  警告=$warnings  总大小=$([math]::Round($totalSizeGB,1))GB" -ForegroundColor Cyan

# 检查快照层数
Write-Host "`n=== 快照层数检查 ===" -ForegroundColor Cyan
Get-VM | ForEach-Object {
    $snaps = Get-Snapshot -VM $_ 2>$null
    $depth = 0
    foreach ($s in $snaps) {
        $chain = 0; $current = $s
        while ($current.ParentSnapshot) { $chain++; $current = $current.ParentSnapshot }
        if ($chain -gt $depth) { $depth = $chain }
    }
    if ($depth -ge $MaxLayers) {
        Write-Host "[WARN] $($_.Name): 快照深度 $($depth+1) 层 (建议 <= $MaxLayers)" -ForegroundColor Red
    }
}

# 检查需要 Consolidate 的 VM
Write-Host "`n=== Consolidation 检查 ===" -ForegroundColor Cyan
Get-VM | Where-Object { $_.ExtensionData.Runtime.ConsolidationNeeded } | ForEach-Object {
    Write-Host "[WARN] $($_.Name): 需要磁盘合并 (Consolidation Needed)" -ForegroundColor Red
}
```

### 5.2 批量清理过期快照

```powershell
# ============================================================
# 批量清理超过指定天数的快照
# 用法：修改参数后运行，默认 DryRun 模式（不实际删除）
# ============================================================

param(
    [int]$MaxAgeDays = 3,
    [switch]$Force  # 不加 -Force 默认只预览不删除
)

$cutoff = (Get-Date).AddDays(-$MaxAgeDays)

$expiredSnaps = Get-VM | Get-Snapshot | Where-Object {
    $_.Created -lt $cutoff
} | Sort-Object @{e={$_.SizeGB};Descending=$true}

if ($expiredSnaps.Count -eq 0) {
    Write-Host "没有超过 $MaxAgeDays 天的快照" -ForegroundColor Green
    return
}

Write-Host "发现 $($expiredSnaps.Count) 个过期快照 (> $MaxAgeDays 天):`n" -ForegroundColor Yellow

$expiredSnaps | Select-Object VM, Name, Created, SizeGB | Format-Table -AutoSize

if ($Force) {
    Write-Host "开始删除..." -ForegroundColor Red
    foreach ($snap in $expiredSnaps) {
        Write-Host "  删除: $($snap.VM.Name) / $($snap.Name) ($([math]::Round($snap.SizeGB,1))GB)..." -NoNewline
        try {
            Remove-Snapshot -Snapshot $snap -Confirm:$false
            Write-Host " OK" -ForegroundColor Green
        } catch {
            Write-Host " FAILED: $_" -ForegroundColor Red
        }
    }
} else {
    Write-Host "[DryRun] 加 -Force 参数实际执行删除" -ForegroundColor Cyan
}
```

### 5.3 邮件告警脚本

```powershell
# ============================================================
# 快照超时邮件告警（配合计划任务每天运行）
# ============================================================

$WarningHours = 24
$SmtpServer = "smtp.company.com"
$From = "vcenter-alert@company.com"
$To = "ops-team@company.com"
$vCenter = "vcenter.company.com"

Connect-VIServer -Server $vCenter -Credential (Import-Clixml "C:\Scripts\vcenter-cred.xml")

$expired = Get-VM | Get-Snapshot | Where-Object {
    $_.Created -lt (Get-Date).AddHours(-$WarningHours)
} | Select-Object VM, Name, Created,
    @{N='AgeDays';E={[math]::Round(((Get-Date) - $_.Created).TotalDays, 1)}},
    @{N='SizeGB';E={[math]::Round($_.SizeGB, 2)}}

if ($expired.Count -gt 0) {
    $body = "<h2>VMware 快照超时告警</h2>"
    $body += "<p>以下快照已超过 $WarningHours 小时，请及时处理：</p>"
    $body += $expired | ConvertTo-Html -Fragment
    $body += "<p style='color:red;'><b>快照存活超过 72 小时会严重影响性能和存储空间！</b></p>"

    Send-MailMessage -From $From -To $To `
        -Subject "[$vCenter] 快照超时告警: $($expired.Count) 个快照需要处理" `
        -Body $body -BodyAsHtml -SmtpServer $SmtpServer
}

Disconnect-VIServer -Confirm:$false
```

---

## 六、紧急处理 Checklist

当快照问题导致生产故障时，按以下顺序处理：

### 存储空间告急（> 90%）

- [ ] 立即检查哪些 VM 有快照：`Get-VM -Datastore "ds1" | Get-Snapshot | Sort SizeGB -Desc`
- [ ] 优先删除最大的、最旧的快照（先删测试环境的）
- [ ] 如果空间不够删快照（合并也需要空间）：
  - Storage vMotion 迁移小 VM 到其他存储
  - 在 VM 内部清理日志/临时文件
  - 临时扩容数据存储（如果是 NFS/iSCSI）
- [ ] 确保可用空间 > 最大 delta 文件大小后再删快照
- [ ] 删除时从最新的快照开始删（减少合并量）

### 快照删除卡住

- [ ] **不要取消正在进行的 consolidate 任务！**
- [ ] 检查 vCenter Tasks 中的任务状态和错误信息
- [ ] 检查存储空间是否足够
- [ ] SSH 到 ESXi 检查日志：
  - `tail -100 /var/log/hostd.log | grep -i snapshot`
  - `tail -100 /var/log/vmkernel.log | grep -i consolidat`
- [ ] 检查是否有磁盘锁冲突：`vmkfstools -D <delta-file>`
- [ ] 尝试 VM → Snapshots → Consolidate
- [ ] 如果 Consolidate 也失败：尝试关机后 Consolidate → 克隆重建（vmkfstools -i）
- [ ] 以上都不行 → 联系 VMware 支持（SR），用 `vm-support` 收集诊断包

### VM 因快照性能下降

- [ ] 检查快照层数（> 3 层影响明显）
- [ ] 检查 delta 文件大小（> 50GB 合并会很慢）
- [ ] 安排维护窗口删除快照
- [ ] 删除前确保有完整备份
- [ ] 在业务低峰期执行删除操作
- [ ] 监控删除过程中的存储延迟（KAVG > 30ms 需关注）

---

## 七、最佳实践总结

| 项目 | 建议 |
|------|------|
| **存活时间** | 不超过 72 小时，24 小时内最佳 |
| **快照层数** | 不超过 3 层，绝对不超过 10 层 |
| **存储预留** | 数据存储至少保留 20% 可用空间 |
| **日常监控** | 每日检查快照状态，超时自动告警 |
| **数据库 VM** | 避免长期保留快照，操作前后立即清理 |
| **备份替代** | 用备份软件（Veeam/NBU）替代快照做长期保留 |
| **操作前** | 确认存储空间充足，确认有完整备份 |
| **操作时** | 选择业务低峰期，避免同时删除多个大快照 |
| **快照命名** | 包含创建人、原因、预期删除时间 |
| **自动化** | 用 PowerCLI 脚本定时检查和清理 |
| **文档记录** | 快照操作记录到变更管理系统 |
| **VMware Tools** | 保持最新版本，避免 VSS 快照问题 |

---

## 八、常见报错速查表

| 报错信息 | 原因 | 处理 |
|---------|------|------|
| `File is locked` | 磁盘被其他进程/主机锁定 | vmkfstools -D 查锁，释放或迁移 |
| `No space left on device` | 数据存储空间不足 | 腾空间后重试 |
| `Cannot open the disk` | delta 文件损坏或路径错误 | 检查文件完整性，尝试 clone 重建 |
| `Consolidation needed` | 快照元数据不一致 | VM → Snapshots → Consolidate |
| `The file specified is not a virtual disk` | vmdk 描述符文件损坏 | 从备份恢复描述符或手动重建 |
| `msg.snapshot.error-quiescingerror` | VSS 快照失败 | 检查 VM 内 VSS 服务，重启 VMware Tools |
| `Too many levels of redo logs` | 快照层数超过 32 层 | 逐层合并，可能需要 clone 重建 |
| `A general system error occurred` | 多种原因 | 查 hostd.log 和 vmkernel.log |
| `The destination file system does not support large files` | VMFS 版本过低 | 升级 VMFS 或迁移到新数据存储 |
| `Operation timed out` | 合并超时（通常是 I/O 慢） | 检查存储性能，等待或在低峰期重试 |

---

## 写在最后

快照是一把双刃剑 — 用好了是临时保险，用不好就是定时炸弹。记住三条铁律：

**1. 快照不是备份 &emsp; 2. 不超过 72 小时 &emsp; 3. 数据库 VM 慎用**

后面还会继续分享：

- **Linux 服务器巡检脚本（Bash 版）**
- **Windows Server 安全加固巡检脚本（PowerShell 版）**
- **VMware ESXi 巡检脚本（PowerCLI 版）** — 即将发布
- **如何搭建轻量级巡检平台**

觉得有用？**点赞 + 收藏 + 转发**，有问题欢迎评论区交流。

---

*本文基于 vSphere 6.7 / 7.0 / 8.0 验证，ESXi 6.5 及更早版本部分命令可能不同。*
