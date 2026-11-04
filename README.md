# stdexec-learn
nvidia的stdexec的学习记录

# 安装

## 环境要求

| 工具 | 最低 | 推荐 | 备注 |
|---|---|---|---|
| gcc / g++ | **12** | 14 | `-std=c++20` 必需；GCC 11 编译不过 |
| clang++ | 16 | 19 | 只在想用 Clang 编译时需要 |
| clangd | **16** | 19 | clangd 前端与 clang 同源，14 会满屏报错 |
| CMake | **3.28** | ≥ 3.28 | 硬门槛，见下方说明 |
| C++ 标准 | 20 | 20 | 只有开 modules 才需要 23 |

> **本机实测通过的组合**（2026-11 已验证）：`g++-14 (14.3.0)` + `cmake 4.4.4` + `clangd 19.1.7`，clangd 解析 stdexec 头文件 `0 errors`。

> **CMake 门槛为什么是 3.28**：本仓库根 `CMakeLists.txt` 写的是 `cmake_minimum_required(VERSION 3.22)`，但 `stdexec/CMakeLists.txt` 第 1 行要求 `3.28.0`。`add_subdirectory` 会采用子项目的要求，所以实际门槛是 **3.28**，用 3.26 会直接停在配置阶段：
> `CMake Error at stdexec/CMakeLists.txt:1: CMake 3.28.0 or higher is required.`

> **GPU 支持**：`nvexec` 需要 `nvc++ 26.5`，**不支持 nvcc**。只是学 CPU 上的 sender/receiver，忽略这一条。

## 部署

### 1. 获取代码（注意 submodule）

`stdexec` 在本仓库里是 **git submodule**（见 `.gitmodules`），不是普通目录。少做一步 `add_subdirectory(stdexec)` 就会失败。

```bash
# 全新克隆：直接带上 submodule
git clone --recursive <本仓库地址>
```

已经 clone 过、`stdexec/` 是空目录的情况：

```bash
git submodule update --init --recursive
```

确认 submodule 到位：

```bash
ls stdexec/include/stdexec/execution.hpp   # 必须存在
git submodule status                        # 前面不能是 '-'
```

目录里真正有用的是：

```
stdexec/include/     # 全部头文件，header-only
stdexec/src/         # parallel_scheduler 的后端实现（可选）
stdexec/test/        # 官方测试，学习时关掉
stdexec/examples/    # 官方示例，学习时关掉
```

### 2. 接入 CMake

`stdexec` 是 header-only 库，接入只需两行：

```cmake
add_subdirectory(stdexec)
target_link_libraries(my_target PRIVATE STDEXEC::stdexec)
```

`STDEXEC::stdexec` 是 INTERFACE target，会把 `-std=c++20`、`-fcoroutines`、`-fconcepts-diagnostics-depth=10` 等编译选项自动带给你。

唯一需要额外链接的是并行调度器后端：见下面「常见坑」第 3 条。

### 3. 配置与构建

```bash
cmake -B build -G Ninja \
  -DCMAKE_CXX_COMPILER=g++-14 \
  -DCMAKE_C_COMPILER=gcc-14 \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DSTDEXEC_BUILD_TESTS=OFF \
  -DSTDEXEC_BUILD_EXAMPLES=OFF

cmake --build build -j$(nproc)
```

几个参数的意义（都很重要）：

- `-DCMAKE_CXX_COMPILER=g++-14`：显式指定编译器，**不要**去改系统默认的 `gcc/g++`，理由见「常见坑」第 4 条。
- `-DSTDEXEC_BUILD_EXAMPLES=OFF`：`STDEXEC_BUILD_EXAMPLES` **默认是 ON**，而且 `examples/` 没有被 `EXCLUDE_FROM_ALL` 排除，所以作为子项目也会被加进构建，首次全量编译几十个示例非常慢。学习时关掉。
- `-DSTDEXEC_BUILD_TESTS=OFF`：子项目模式下默认就是 OFF（只有 stdexec 作为顶层项目时才默认 ON），写出来只是为了明确。想看官方测试就改 ON，但它依赖 Relacy（需 GCC 12+）且编译极慢。
- `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`：生成 `build/compile_commands.json`，clangd 用。
- `-DCMAKE_BUILD_TYPE=Release`：Debug 会踩到 TBB 链接问题（见「常见坑」第 6 条），Release 靠 `-O3` 能过；CMake 找得到系统 TBB 时也会自动链接上去。

> **首次 configure 需要联网**：`stdexec/cmake/RAPIDS.cmake` 会从 GitHub 拉取 `rapids-cmake`（默认 `branch-24.02`）。离线环境下配置会卡在这里，不是你的 CMake 版本问题。

### 4. 验证部署是否成功

```bash
cat > /tmp/smoke.cpp <<'EOF'
#include <stdexec/execution.hpp>
#include <cstdio>

namespace ex = stdexec;

int main() {
  auto [v] = ex::sync_wait(ex::just(42)).value();
  std::printf("%d\n", (int)v);   // 期望输出 42
}
EOF

g++-14 -std=c++20 -D_GLIBCXX_USE_TBB_PAR_BACKEND=0 \
  -I stdexec/include /tmp/smoke.cpp -o /tmp/smoke
./smoke
```

输出 `42` 说明头文件、编译器、标准都对上了。这一步不经过 CMake，能快速把「工具链问题」和「CMake 配置问题」区分开。

`-D_GLIBCXX_USE_TBB_PAR_BACKEND=0` **不是可选项**：stdexec 内部会 `#include <execution>`，GCC 14 的 libstdc++ 默认开着 TBB 并行后端，不加这个宏会链接失败。原因见「常见坑」第 6 条。

### 5. clangd 配置

`stdexec` 是纯头文件库，`compile_commands.json` 里**没有** stdexec 头文件的编译命令（`stdexec/CMakeLists.txt:631` 那个 clangd helper 走的是 `STDEXEC_IS_TOP_LEVEL` 分支，本仓库里没有定义，所以不会生成）。因此 clangd 找不到 `-I`，写代码时会报 `'stdexec/execution.hpp' file not found`。

在仓库根新建 `.clangd`，把 include 路径补上：

```yaml
CompileFlags:
  Add:
    - -std=c++20
    - -I/home/kepler/stdexec-learn/stdexec/include
```

```bash
cat > /home/kepler/stdexec-learn/.clangd <<'EOF'
CompileFlags:
  Add:
    - -std=c++20
    - -I/home/kepler/stdexec-learn/stdexec/include
EOF
```

`stdexec/.clangd` 里官方已经配好了 `-std=c++20`、`-DSTDEXEC_CLANGD_INVOKED`、剥离 `-fcoroutines` 和 CUDA 参数的逻辑，**不要改它**，只在根目录补 `-I` 即可。

这个 `-I` 是绝对路径，属于本机专用配置，**不要提交**。本仓库已把它加到本地排除文件：

```bash
grep -q "^\\.clangd$" .git/info/exclude || echo ".clangd" >> .git/info/exclude
git check-ignore -v .clangd     # 应该能看到匹配规则
```

不开编辑器也能验证 clangd 是否真能解析 stdexec 头文件：

```bash
mkdir -p /tmp/ccdb && cat > /tmp/ccdb/compile_commands.json <<'EOF'
[{"directory":"/tmp","file":"/tmp/smoke.cpp",
  "command":"g++-14 -std=c++20 -I/home/kepler/stdexec-learn/stdexec/include -c /tmp/smoke.cpp"}]
EOF

clangd-19 --check=/tmp/smoke.cpp --compile-commands-dir=/tmp/ccdb 2>&1 | tail -1
```

期望最后一行是 `All checks completed, 0 errors`（实测建 preamble 约 29 MB、1.2 秒）。如果报 `'stdexec/execution.hpp' file not found`，就是 `.clangd` 里的 `-I` 没生效。

## 编译工具链的更新

编译 stdexec 的底线是 `gcc/g++ 12 + cmake 3.28`，如果达不到标准需要先更新再编译。

推荐的组合为：`g++-14 + clangd-19 + cmake ≥ 3.28`

### gcc / g++ 14

Ubuntu 22.04 官方源没有 GCC 14，走 ubuntu-toolchain-r PPA：

```bash
sudo add-apt-repository -y ppa:ubuntu-toolchain-r/test
sudo apt update
sudo apt install -y gcc-14 g++-14

gcc-14 --version | head -1
g++-14 --version | head -1
```

### cmake ≥ 3.28

用 pip 装到用户目录，不动系统 cmake：

```bash
python3 -m pip install --user -U "cmake>=3.28"
hash -r
cmake --version | head -1      # 期望 ≥ 3.28
which cmake                    # 期望 /home/<user>/.local/bin/cmake
```

如果 `which cmake` 仍指向 `/usr/local/bin/cmake`（3.26）：

```bash
export PATH="$HOME/.local/bin:$PATH"   # 写进 ~/.bashrc
rm -f /usr/local/bin/cmake             # 或直接删掉旧的 3.26，需要 sudo
```

### clangd 19

Ubuntu 22.04 源里最高只有 clangd 14，必须加 LLVM 官方源。

> **国内没有可用的 llvm-apt 完整镜像**（实测）：清华 / 北外返回 403；南大只同步了 dists 索引，pool 里的 deb 全 404；USTC / 阿里 / 腾讯没有这个仓库；华为云返回的是网页不是镜像。所以只能直连 `apt.llvm.org`，**而且必须直连**：走 `127.0.0.1:7897` 这类本地代理会把 pool 请求打成 404。

```bash
# 1. 加源（只加 amd64，省掉 i386 索引）
curl -fsSL https://apt.llvm.org/llvm-snapshot.gpg.key \
  | sudo tee /etc/apt/trusted.gpg.d/apt.llvm.org.asc > /dev/null

echo "deb [arch=amd64] http://apt.llvm.org/jammy/ llvm-toolchain-jammy-19 main" \
  | sudo tee /etc/apt/sources.list.d/llvm-19.list > /dev/null

# 2. 只更新这两个源（关键：直接 apt update 会挨个等 docker / vscode / ros 等十几个国外源超时）
sudo apt-get update -o Dir::Etc::sourceparts="-" \
  -o Dir::Etc::sourcelist="/etc/apt/sources.list" -o APT::Get::List-Cleanup="0"
sudo apt-get update -o Dir::Etc::sourceparts="-" \
  -o Dir::Etc::sourcelist="/etc/apt/sources.list.d/llvm-19.list" -o APT::Get::List-Cleanup="0"

# 3. 安装：跳过 Recommends 里的 libclang-rt-19-dev（省 3.8 MB），总共约 45 MB
sudo apt-get install -y --no-install-recommends clangd-19

clangd-19 --version | head -1     # 期望 LLVM version 19.1.7
```

apt **不读** `http_proxy` 环境变量，它走的是直连，实测 271 kB/s（套本地代理只有 186 kB/s，反而更慢），所以不要给 apt 配代理。

**如果还是太慢**，可以断点续传手动下 4 个 deb 再本地安装（`wget -c` 中断后重跑同一条命令即可续传）：

```bash
mkdir -p ~/clangd19-deb && cd ~/clangd19-deb
V='19.1.7~++20250114103320+cd708029e0b2-1~exp1~20250114103432.75'
B='http://apt.llvm.org/jammy/pool/main/l/llvm-toolchain-19'
for p in libllvm19 libclang-cpp19 libclang-common-19-dev clangd-19; do
  wget --no-proxy -c "$B/${p}_${V}_amd64.deb" || break
done
sudo apt-get install -y --no-install-recommends ./*.deb
```

Ubuntu 主源换成阿里云能顺带快一点（拉 `g++-12` 12 MB 实测：阿里云 444 kB/s、sysu 391 kB/s、USTC 318 kB/s、清华 403 被拦）：

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak.$(date +%F)
sudo sed -i 's|http://mirror.sysu.edu.cn/ubuntu/*|https://mirrors.aliyun.com/ubuntu|g' /etc/apt/sources.list
```

让默认 `clangd` 指向 19（保留 `/usr/bin/clangd -> clangd-14` 不动，只影响当前用户）：

```bash
mkdir -p ~/.local/bin
ln -sf /usr/bin/clangd-19 ~/.local/bin/clangd
hash -r && which clangd && clangd --version | head -1
```

前提是 `~/.local/bin` 在 `PATH` 里排在 `/usr/bin` 前面（本机已满足）。本机的 `/usr/bin/clangd` 仍然指向 14，其他工具不受影响。

编辑器里也要显式指过去，否则还在用旧的 14：

- VS Code：`clangd.path` → `/usr/bin/clangd-19`
- Neovim / Emacs：`clangd` → `/usr/bin/clangd-19`

## 常见坑

### 1. 配置阶段就报 CMake 版本不够

```
CMake Error at stdexec/CMakeLists.txt:1 (cmake_minimum_required):
  CMake 3.28.0 or higher is required.  You are running version 3.26.0
```

按上面「cmake ≥ 3.28」升级，并确认 `which cmake` 指向的是新版。

### 2. 编译器太老，报一堆看不懂的模板错误

GCC 11 的典型报错（不是你的代码问题）：

```
__just.hpp:166: error: deduction guide 'just_t::__sender(...)'
                must be declared at namespace scope
```

Clang 14 的典型报错：

```
__config.hpp:44: fatal error: 'ciso646' file not found
```

两者都只需要升到 GCC 12+ / Clang 16+ 即可消失。

### 3. `get_parallel_scheduler()` 链接通过但运行时崩溃

官方 README 明确警告：`get_parallel_scheduler()` 需要把后端编译进来，否则**能链接、能跑，但调用时崩溃**。两条路选一条：

```cmake
# 方案 A：编译 stdexec 的后端，并额外链接
set(STDEXEC_BUILD_PARALLEL_SCHEDULER ON)
target_link_libraries(my_target PRIVATE STDEXEC::stdexec STDEXEC::parallel_scheduler)
```

```cpp
// 方案 B：头文件内联实现，不用额外链接
#define STDEXEC_PARALLEL_SCHEDULER_HEADER_ONLY
```

学习阶段用 `ex::get_parallel_scheduler()`，走方案 B 最省事。

### 4. 不要用 `update-alternatives` 改系统默认 gcc

```bash
# 千万别做
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-14 100
```

本机装着 `/opt/ros/humble`，ROS 2 Humble 是按 GCC 11 的头文件/ABI 编译的，换掉系统默认编译器会连带炸掉 ROS 的构建。正确做法是**按项目**指定：`-DCMAKE_CXX_COMPILER=g++-14`，或者临时 `CC=gcc-14 CXX=g++-14 cmake ...`。

### 5. 想跑 stdexec 的模块（modules）构建

`-DSTDEXEC_BUILD_MODULES=ON` 会切到 **C++23**，并且需要很新的 Clang（官方 CI 用的是 clang 22 / trunk）。日常学习用不到。

### 6. 链接失败：`undefined reference to tbb::detail::r1::execution_slot`

`stdexec/__detail/__execution_legacy.hpp` 里 `#include <execution>`，GCC 14 的 libstdc++ 默认开启 TBB 并行后端，于是链接期要找 TBB：

```
undefined reference to `tbb::detail::r1::execution_slot(tbb::detail::d1::execution_data const*)'
```

要分四种情况看：

| 构建方式 | 结果 |
|---|---|
| 直接 `g++-14 -O0 ...` | **链接失败**，必须 `-ltbb` 或 `-D_GLIBCXX_USE_TBB_PAR_BACKEND=0` |
| 直接 `g++-14 -O3 ...` | 侥幸能过（没被用到的 inline 被丢掉），别依赖 |
| CMake Debug，系统有 TBB | 能过：`cmake/Modules/ConfigureTBB.cmake` 发现 `<tbb/tbb.h>` + libstdc++ 就自动链接 |
| CMake Debug，系统没有 TBB | **链接失败**，同上 |

稳妥做法：统一加 `-D_GLIBCXX_USE_TBB_PAR_BACKEND=0`（stdexec 自己在 32 位 GCC 14 构建里也这么干），或在 CMake 里显式 `target_link_libraries(my_target PRIVATE tbb)`。
