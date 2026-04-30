# Ansible 磁盘自动挂载 Playbook

批量格式化和挂载磁盘到远程主机，支持 EXT4 和 XFS 两种文件系统，使用 UUID 方式写入 fstab 确保重启后稳定挂载。

## 前置条件

- 已安装 Ansible
- 控制节点可通过 SSH 免密登录目标主机
- 关闭首次 SSH 登录的 host key 检查（可选）：

```bash
# 编辑 /etc/ansible/ansible.cfg
[defaults]
host_key_checking = False
```

## 文件说明

| 文件 | 说明 |
|------|------|
| `hosts` | Inventory 文件，定义目标主机组 `stageall` |
| `automount.yml` | EXT4 格式化与挂载（适用于 NVMe 等整盘直接使用的场景） |
| `mount_xfs.yml` | XFS 分区、格式化与挂载（适用于需要 GPT 分区表的场景，如 virtio 磁盘） |

## Inventory 配置

编辑 `hosts` 文件，将 IP 替换为实际目标主机地址：

```ini
[stageall]
10.48.0.101
10.48.0.102
10.48.0.103
```

## Playbook 1：EXT4 整盘挂载（automount.yml）

### 功能

1. 创建挂载目录
2. 卸载已有挂载并清理 fstab 旧条目
3. 强制格式化磁盘为 EXT4
4. 通过 `blkid` 获取 UUID
5. 使用 UUID 挂载并写入 fstab

### 变量配置

编辑 `automount.yml` 中的 `disks` 字典，key 为设备路径，value 为挂载点：

```yaml
vars:
  disks:
    /dev/nvme1n1: /data
    # 可添加多块盘
    # /dev/nvme2n1: /data2
```

### 执行

```bash
ansible-playbook -i hosts automount.yml
```

> ⚠️ **警告**：此 playbook 使用 `force: true` 强制格式化，每次运行都会清除磁盘数据，请确认目标磁盘无重要数据。

## Playbook 2：XFS 分区挂载（mount_xfs.yml）

### 功能

1. 更新 apt 缓存并安装 `xfsprogs`
2. 使用 `parted` 创建 GPT 分区表和分区
3. 将分区格式化为 XFS
4. 创建挂载目录
5. 通过 `blkid` 获取分区 UUID
6. 使用 UUID 挂载并写入 fstab

### 变量配置

编辑 `mount_xfs.yml` 中的变量：

```yaml
vars:
  mount_point: /data        # 挂载目录
  device_name: /dev/vdb     # 磁盘设备
  partition_name: /dev/vdb1  # 分区设备（parted 创建后的分区路径）
```

### 执行

```bash
ansible-playbook -i hosts mount_xfs.yml
```

> 📌 **注意**：此 playbook 使用 `apt` 安装依赖，仅适用于 Debian/Ubuntu 系统。

## 两个 Playbook 的区别

| | automount.yml | mount_xfs.yml |
|---|---|---|
| 文件系统 | EXT4 | XFS |
| 分区方式 | 不分区，直接使用整盘 | GPT 分区表 + 分区 |
| 多盘支持 | 支持（字典扩展） | 单盘（需修改变量） |
| 目标系统 | 通用 Linux | Debian/Ubuntu |
| 典型场景 | NVMe 数据盘 | 云主机 virtio 磁盘 |
