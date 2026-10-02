# Obsidian Git 操作记录

## 1. 基础环境检查

```
git --version
git config --global user.name
git config --global user.email
```

## 2. 连接远程仓库

```
cd "E:\obsidian\我的记录"
git remote add origin https://github.com/xxooxxooxx/Obsidian
git push -u origin master
```

> 如果分支是 main，把 master 改成 main。首次推送建议带 -u 建立上游跟踪。

## 3. 修复 .gitignore 编码（关键）

Git 只识别 **UTF-8（无 BOM）** 的 .gitignore 文件。UTF-16 BOM 会导致忽略规则完全无法解析。

### 转换为 UTF-8（无 BOM）

```
cd "E:\obsidian\我的记录"
python -c "
import codecs
content = codecs.open('.gitignore', 'r', 'utf-16-le').read()
with codecs.open('.gitignore', 'w', 'utf-8') as f:
    f.write(content)
"
```

### 验证编码是否正确

```
cd "E:\obsidian\我的记录"
python -c "d=open('.gitignore','rb').read(); print(repr(d[:15]))"
```

**预期结果：** 以 '.obsidian/' 开头，不要出现 \xef\xbb\xbf（BOM）。

### 验证忽略规则是否生效

```
cd "E:\obsidian\我的记录"
git check-ignore -v .gitignore
git check-ignore -v .obsidian
git status --short
```

**预期结果：**
- git check-ignore -v 能正确匹配到对应规则
- git status --short **无输出**（说明被忽略的未跟踪文件未列出）
- git status --short --ignored 会显示 !! .gitignore 和 !! .obsidian/

## 4. 正确配置 .gitignore（避免被重新提交）

要让 .obsidian/ 和 .gitignore 永远不被 Obsidian Git 的 Commit All（git add .）重新提交上去，必须在 .gitignore 里把它们自己也忽略掉。

### 推荐的 .gitignore 内容

```
.obsidian/
*.log
.DS_Store
Thumbs.db
.gitignore
```

> **注意：** .gitignore 不需要提交到仓库也能生效。只要编码是 UTF-8（无 BOM）且文件存在于本地工作目录，Git 就能直接读取并生效。

## 5. 清理历史记录（移除 .obsidian/）

如果远程仓库历史里还残留了 .obsidian/ 文件，需要用 git-filter-repo 重写整个历史。

### 5.1 安装 git-filter-repo

```
python -m pip install git-filter-repo
```

### 5.2 备份（强烈推荐）

在操作前，**务必整份复制一份整个仓库文件夹**作为备份。

### 5.3 重写所有分支和标签的历史

```
cd "E:\obsidian\我的记录"
python -m git_filter_repo --path .obsidian --invert-paths --force --replace-refs update-or-add
```

> git-filter-repo 会在重写历史后自动移除 origin 远程，这是正常安全行为。

### 5.4 重新添加远程并强制推送

```
cd "E:\obsidian\我的记录"
git remote add origin https://github.com/xxooxxooxx/Obsidian
git push origin --force --all
git push origin --force --tags
```

### 5.5 验证是否清理干净

最可靠的验证方式是**重新克隆到临时目录**检查：

```
cd "C:\Temp"
git clone https://github.com/xxooxxooxx/Obsidian obsidian-clean
cd obsidian-clean
git rev-list --objects --all -- .obsidian
```

**预期结果：** 没有任何输出，说明 .obsidian/ 已完全从远程历史中清除。

## 6. 清理历史记录（移除 .gitignore，可选）

如果想连历史里的 .gitignore 也彻底清除（只影响历史，不影响本地文件）：

```
cd "E:\obsidian\我的记录"
python -m git_filter_repo --path .gitignore --invert-paths --force --replace-refs update-or-add
git remote add origin https://github.com/xxooxxooxx/Obsidian
git push origin --force --all
git push origin --force --tags
```

验证同上，把 .obsidian 换成 .gitignore。

## 7. 清理最近的提交（不重写全历史）

如果只想合并最近几个提交（squash），可以用交互式 rebase：

```
cd "E:\obsidian\我的记录"
git rebase -i HEAD~3
```

把需要合并的 commit 前的 pick 改成 squash（或 s），保留一个主 commit。完成后正常推送即可（如果已经推送过需要 git push --force-with-lease）。

## 8. 重要注意事项

- **历史重写（git-filter-repo）** 会改变所有 Commit SHA。操作前**必须备份整个仓库**。
- **强制推送后，所有协作者都必须重新克隆**，旧本地仓库与新远程历史不兼容。
- **分支保护**：如果 GitHub 开启了分支保护，需要临时允许 Force Push，推送完成后再关闭。
- **.gitignore 是否需要提交**：不需要。Git 直接读取本地 .gitignore 即可生效，只有想要把忽略规则共享给其他协作者时才需要提交。
- **编码是关键**：.gitignore 必须是 **UTF-8（无 BOM）**，否则规则不会生效。
- **Git glob 对中文名支持有限**：要忽略整个目录（如 TEST/）比用通配符匹配中文文件名更稳定可靠。

