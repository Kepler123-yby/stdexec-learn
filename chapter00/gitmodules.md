# .gitmodules 使用说明

> 本仓库已经把 [NVIDIA/stdexec](https://github.com/NVIDIA/stdexec) 作为 submodule 引入，
> 钉在提交 `1d58877`。这份笔记记录它的原理和常用操作，方便以后查阅。

## 1. 四个位置，别搞混

```
stdexec-learn/                      ← 父仓库（superproject）
├── .gitmodules                     ← ① 声明：path + url，会提交、会分发
├── .git/
│   ├── config                      ← ③ 本机私有：submodule.stdexec.url / .active
│   └── modules/stdexec/            ← ④ 子模块真正的 git 仓库（本地缓存）
├── README.md                       ← 普通文件（内容存进树里）
└── stdexec/
    └── .git                        ← 只是个文本文件："gitdir: ../.git/modules/stdexec"
```

| 位置 | 内容 | 进仓库？ | 作用 |
| --- | --- | --- | --- |
| ① `.gitmodules` | `path` / `url` / `branch` | ✅ 提交 | 声明依赖谁、放在哪 |
| ② 父仓库 tree | `160000 <commit> stdexec` | ✅ 提交 | **钉死版本** |
| ③ `.git/config` | `submodule.stdexec.url` 等 | ❌ 本地 | **覆盖** `.gitmodules`（优先级更高） |
| ④ `.git/modules/stdexec` | 完整子仓库对象 | ❌ 本地 | 真正干活的对象库 |

关键结论：

- `.gitmodules` 里**没有版本号**，版本在父仓库的 gitlink（`mode 160000`）里。
- 改 ③ 只影响自己这台机器；改 ① 并提交才会影响所有人（`git submodule sync` 把 ① 同步到 ③）。
- 子模块内容不进父仓库的对象库，父仓库只记了"那个仓库的哪个提交"。

## 2. .gitmodules 的字段

```ini
[submodule "stdexec"]                              # 子模块名，默认等于 path
	path = stdexec                                 # 必填：检出到父仓库的哪个目录
	url = https://github.com/NVIDIA/stdexec.git    # 必填：从哪拉

	# 以下均为可选
	branch = main              # update --remote 跟哪个分支（默认跟远端 HEAD）
	update = rebase            # 默认更新策略：checkout / merge / rebase / none
	ignore = dirty             # status / diff 是否提示脏状态：all / dirty / untracked / none
	shallow = true             # 浅克隆该子模块（git 2.30+）
	fetchRecurseSubmodules = true   # 见下节"嵌套子模块"
```

嵌套子模块（子模块自己还有子模块）时，最外层需要 `fetchRecurseSubmodules = true`，
并且操作时加 `--recursive`。

## 3. 命令清单

### 取代码

```bash
git clone --recurse-submodules git@github.com:Kepler123-yby/stdexec-learn.git
git submodule update --init --recursive      # 已经 clone 过，事后补拉
git submodule update --init stdexec          # 只初始化某一个
```

### 查看状态

```bash
git submodule status
#   无前缀  与父仓库记录的提交一致
#   -       未初始化（目录是空的）
#   +       子模块当前提交 ≠ 父仓库记录的提交
#   U       合并冲突

git submodule summary          # 子模块里有哪些提交差异
git submodule foreach 'git status -sb'   # 在每个子模块内执行同一条命令
```

### 更新版本

```bash
# 方式一：拉 .gitmodules 里 branch 的最新提交
git submodule update --remote --merge stdexec

# 方式二：手动钉到某个 tag / commit
git -C stdexec fetch
git -C stdexec checkout <tag 或 commit>

# 两种方式最后都必须更新父仓库的 gitlink，否则别人拉到的还是旧版本
git add stdexec
git commit -m "chore: bump stdexec"
git push
```

### 维护

```bash
git submodule set-url stdexec <新url>        # 改 .gitmodules（git 2.25+）
git submodule set-branch --branch main stdexec
git submodule sync --recursive               # 把 .gitmodules 的 url 同步进 .git/config
git submodule deinit -f stdexec              # 从工作树撤除，并清掉 .git/config 条目
git submodule absorbgitdirs                  # 把子模块的 .git 收进 .git/modules/
```

## 4. 日常流程

### 本机配置一次：让 pull 自动带子模块

```bash
git config --global submodule.recurse true
# 之后 git pull 等价于 git pull --recurse-submodules
# 不配的话，每次要写 git pull --recurse-submodules
```

### 拉取别人的更新

```bash
git pull                # 已配 submodule.recurse 时，子模块一起更新
git pull --recurse-submodules   # 未配时
```

### 在本仓库里改 stdexec 代码？

**不行，除非换成自己的 fork。** 子模块的 url 指向 NVIDIA 官方仓库，
你无法推送，所以在 `stdexec/` 里的提交无法通过本仓库分发。要改源码只有两条路：

```bash
# 路线 A：fork + 换 url
# 1. GitHub 上 fork NVIDIA/stdexec
git submodule set-url stdexec https://github.com/<你>/stdexec.git
git submodule sync
git -C stdexec switch main          # 一定要先切回分支，否则是 detached HEAD
# 2. 改代码、在 stdexec 里 commit、push 到你的 fork
# 3. 父仓库里记录新提交
git add stdexec && git commit -m "chore: 指向自己的 stdexec fork" && git push

# 路线 B：vendor（不要子模块，源码直接进仓库）
git submodule deinit -f stdexec
git rm --cached stdexec
rm -rf stdexec .gitmodules
git clone --depth 1 https://github.com/NVIDIA/stdexec.git stdexec
rm -rf stdexec/.git
git add stdexec && git commit -m "chore: vendor stdexec"
```

### 只想临时试新版、不想提交

```bash
git -C stdexec checkout <某个提交>
git submodule status        # 会显示 + ...，表示与父仓库记录不一致
# 不执行 git add stdexec，就不会被记录
```

## 5. 五个常见的坑

1. **clone 忘了 `--recurse-submodules`** → `stdexec/` 是空目录。
   补救：`git submodule update --init --recursive`。

2. **在子模块里 commit 了，父仓库没 `git add`**
   → 别人 `submodule update` 时报
   `fatal: remote error: upload-pack: not our ref` 或
   `Server does not allow request for unadvertised object`。
   因为父仓库钉的提交只存在于你本地，上游服务器上没有。

3. **子模块默认是 detached HEAD**
   直接在 `stdexec/` 里提交是"游离提交"，一次 `git submodule update` 就丢了。
   要先 `git -C stdexec switch main`（或自建分支）再改。

4. **改了 `.gitmodules` 却不生效**
   本机 `.git/config` 里的覆盖值还在，需要 `git submodule sync`。

5. **父仓库显示 `+` 但不知道差在哪**
   `git submodule summary` 看具体提交差异；`git -C stdexec status` 看工作区脏否。

## 6. 和 .gitignore 的区别

| | `.gitignore` | `.gitmodules` |
| --- | --- | --- |
| 作用 | 让 git **看不见**这些文件 | 告诉 git 这是个**外部仓库**，并钉版本 |
| 结果 | 文件不进仓库 | 只进一个 gitlink + 声明 |
| 更新方式 | 无 | 需要 `submodule update --init` 才有内容 |

两者不冲突。`stdexec/` 现在能进仓库（存指针），而 `build/` 被 `.gitignore` 排除。

## 7. 本仓库现状速查

```bash
git submodule status
#  1d58877ad898fa3422658d47b28f67deb9c10e80 stdexec (heads/main)
cat .gitmodules
#  [submodule "stdexec"]
#  	path = stdexec
#  	url = https://github.com/NVIDIA/stdexec.git
git ls-files -s stdexec
#  160000 1d58877ad898fa3422658d47b28f67deb9c10e80 0	stdexec
```

构建时把它当普通 CMake 子项目用（见 [install.md](install.md)）：

```cmake
add_subdirectory(stdexec)
target_link_libraries(my_target PRIVATE STDEXEC::stdexec)
```

> 注意：本机拉取 stdexec 走 `https://github.com`，需要代理；
> 已在 `~/.bashrc` 里把 GitHub 从 `no_proxy` 中移除，新终端生效。
