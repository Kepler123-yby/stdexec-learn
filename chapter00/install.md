# 安装

## 部署

```bash
git clone https://github.com/NVIDIA/stdexec.git
```

### 配置：

```cmake
add_subdirectory(stdexec)
target_link_libraries(my_target PRIVATE STDEXEC::stdexec)
```