# 为 Codex 安装 Superpowers

通过原生技能发现启用 superpowers 技能在 Codex 中。只需 clone 和 symlink。

## 前提条件

- Git

## 安装

1. **克隆 superpowers 仓库：**
   ```bash
   git clone https://github.com/obra/superpowers.git ~/.codex/superpowers
   ```

2. **创建技能 symlink：**
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/superpowers/skills ~/.agents/skills/superpowers
   ```

   **Windows (PowerShell)：**
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
   cmd /c mklink /J "$env:USERPROFILE\.agents\skills\superpowers" "$env:USERPROFILE\.codex\superpowers\skills"
   ```

3. **重启 Codex**（退出并重新启动 CLI）以发现技能。

## 从旧 bootstrap 迁移

如果你在原生技能发现前安装了 superpowers，你需要：

1. **更新 repo：**
   ```bash
   cd ~/.codex/superpowers && git pull
   ```

2. **创建技能 symlink**（上面步骤 2）— 这是新的发现机制。

3. **从 `~/.codex/AGENTS.md` 移除旧 bootstrap 块** — 任何引用 `superpowers-codex bootstrap` 的块不再需要。

4. **重启 Codex。**

## 验证

```bash
ls -la ~/.agents/skills/superpowers
```

你应该看到指向你的 superpowers 技能目录的 symlink（或 Windows 上是 junction）。

## 更新

```bash
cd ~/.codex/superpowers && git pull
```

技能通过 symlink 立即更新。

## 卸载

```bash
rm ~/.agents/skills/superpowers
```

可选删除克隆：`rm -rf ~/.codex/superpowers`。