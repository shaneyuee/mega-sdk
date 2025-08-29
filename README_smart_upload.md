# Smart Upload with MAC-based Deduplication

## Overview

This document describes the implementation of smart upload functionality that uses MAC (Message Authentication Code) verification to avoid redundant file uploads when the same file content already exists in the cloud.

## How It Works

### 1. File Fingerprint Detection
When a file is queued for upload, the system first generates a fingerprint of the local file and checks if any existing cloud file has the same fingerprint.

**Key Code Location**: `src/megaclient.cpp` in `startxfer()` function
```cpp
std::shared_ptr<Node> samenode = mNodeManager.getNodeByFingerprint(*f);
if (samenode && samenode->nodekey().size())
{
    // Found a file with same fingerprint
}
```

### 2. MAC Verification for Security
If a file with the same fingerprint is found, the system performs MAC verification to ensure the files are truly identical and not just hash collisions.

**Key Code Location**: `src/megaclient.cpp`
```cpp
bool macVerified = CompareLocalFileMetaMacWithNode(fa.get(), samenode.get());
if (macVerified)
{
    LOG_info << "MAC verification successful for file deduplication";
}
```

**Important**: The MAC verification uses the existing utility function `CompareLocalFileMetaMacWithNode()` from `src/utils.cpp` to ensure cryptographic integrity.

### 3. Node Duplication (Deduplication)
When MAC verification succeeds, instead of uploading the file data again, the system creates a new node that references the same file content.

**Key Code Location**: `src/megaclient.cpp`
```cpp
// Create a new node that references the same file data
TreeProcCopy tc;
proctree(samenode, &tc, false, true);
tc.allocnodes();
proctree(samenode, &tc, false, true);

// CRITICAL: Set handles correctly
tc.nn[0].parenthandle = UNDEF;  // Let putnodes handle parent

// Submit the new node creation request
putnodes(f->h, UseLocalVersioningFlag, std::move(tc.nn), nullptr, tag, false);
```

### 4. Critical Implementation Details

#### Parameter Handling in putnodes()
**CRITICAL**: When calling `putnodes()`, certain parameters must NOT be pre-assigned to the File object:

```cpp
// ❌ WRONG - This causes API_ENOENT (-9) error
f->tag = tag;
putnodes(f->h, UseLocalVersioningFlag, std::move(tc.nn), nullptr, f->tag, false);

// ✅ CORRECT - Pass tag directly without assigning to f->tag
putnodes(f->h, UseLocalVersioningFlag, std::move(tc.nn), nullptr, tag, false);
```

#### Node Handle Management
- `tc.nn[0].parenthandle = UNDEF` - Let putnodes function set the parent
- Use `nodehandle` and the same `nodekey` from the source node to reference the same file content

### 5. File Structure and Key Components

```
src/megaclient.cpp     - Main implementation in startxfer()
src/utils.cpp          - MAC verification utilities
include/mega/megaclient.h - Function declarations
```

### 6. Testing with megacli

You can test the smart upload functionality using the megacli tool:

```bash
# Upload the same file to two different locations
./megacli
  put /path/to/file /remote/folder1/
  put /path/to/file /remote/folder2/

# The second upload should trigger deduplication
# Check logs for "MAC verification successful" messages
```

#### Expected Log Output
- First upload: Normal upload process
- Second upload: Should show MAC verification and node duplication
- Look for messages like:
  ```
  MAC verification successful for file deduplication
  Somenode having exactly the same MAC exists
  ```

### 7. Error Scenarios and Debugging

#### Common Error Codes
- `-2` (API_EARGS): Invalid arguments, check parameter passing
- `-9` (API_ENOENT): Resource not found, often caused by incorrect tag assignment
- `-14` (API_EKEY): Cryptographic error, check nodehandle validity

#### Debug Logging
The implementation includes extensive debug logging:
```cpp
LOG_debug << "Putnodes for deduplication: parent=" << toNodeHandle(f->h) 
          << " filename=" << sname << " nodekey_size=" << tc.nn[0].nodekey.size();
```

### 8. Benefits

1. **Bandwidth Savings**: Avoids re-uploading identical file content
2. **Time Efficiency**: Faster "upload" completion for duplicate files
3. **Storage Optimization**: Server-side deduplication reduces storage usage
4. **Security**: MAC verification ensures cryptographic integrity

---

# 智能上传与基于MAC的去重功能

## 概述

本文档描述了智能上传功能的实现，该功能使用MAC（消息认证码）验证来避免在云端已存在相同文件内容时进行冗余的文件上传。

## 工作原理

### 1. 文件指纹检测
当文件排队上传时，系统首先生成本地文件的指纹，并检查是否有现有云文件具有相同的指纹。

**关键代码位置**: `src/megaclient.cpp` 中的 `startxfer()` 函数
```cpp
std::shared_ptr<Node> samenode = mNodeManager.getNodeByFingerprint(*f);
if (samenode && samenode->nodekey().size())
{
    // 找到具有相同指纹的文件
}
```

### 2. MAC验证确保安全性
如果找到具有相同指纹的文件，系统会执行MAC验证以确保文件真正相同，而不仅仅是哈希碰撞。

**关键代码位置**: `src/megaclient.cpp`
```cpp
bool macVerified = CompareLocalFileMetaMacWithNode(fa.get(), samenode.get());
if (macVerified)
{
    LOG_info << "MAC verification successful for file deduplication";
}
```

**重要说明**: MAC验证使用了来自 `src/utils.cpp` 的现有工具函数 `CompareLocalFileMetaMacWithNode()` 来确保密码学完整性。

### 3. 节点复制（去重）
当MAC验证成功时，系统不会再次上传文件数据，而是创建一个引用相同文件内容的新节点。

**关键代码位置**: `src/megaclient.cpp`
```cpp
// 创建引用相同文件数据的新节点
TreeProcCopy tc;
proctree(samenode, &tc, false, true);
tc.allocnodes();
proctree(samenode, &tc, false, true);

// 关键：正确设置句柄
tc.nn[0].parenthandle = UNDEF;  // 让putnodes处理父节点

// 提交新节点创建请求
putnodes(f->h, UseLocalVersioningFlag, std::move(tc.nn), nullptr, tag, false);
```

### 4. 关键实现细节

#### putnodes()中的参数处理
**关键点**: 调用 `putnodes()` 时，某些参数不能预先分配给File对象：

```cpp
// ❌ 错误 - 这会导致API_ENOENT (-9)错误
f->tag = tag;
putnodes(f->h, UseLocalVersioningFlag, std::move(tc.nn), nullptr, f->tag, false);

// ✅ 正确 - 直接传递tag而不分配给f->tag
putnodes(f->h, UseLocalVersioningFlag, std::move(tc.nn), nullptr, tag, false);
```

#### 节点句柄管理
- `tc.nn[0].parenthandle = UNDEF` - 让putnodes函数设置父节点
- 使用源节点的 `nodehandle` 和相同的 `nodekey` 来引用相同的文件内容

### 5. 文件结构和关键组件

```
src/megaclient.cpp     - startxfer()中的主要实现
src/utils.cpp          - MAC验证工具
include/mega/megaclient.h - 函数声明
```

### 6. 使用megacli测试

您可以使用megacli工具测试智能上传功能：

```bash
# 将同一文件上传到两个不同位置
./megacli
  cd /remote/folder1/
  put /path/to/file
  cd /remote/folder2/
  put /path/to/file

# 第二次上传应该触发去重
# 检查日志中的"MAC verification successful"消息
```

#### 预期日志输出
- 第一次上传：正常上传过程
- 第二次上传：应显示MAC验证和节点复制
- 查找类似以下的消息：
  ```
  MAC verification successful for file deduplication
  Somenode having exactly the same MAC exists
  ```

### 7. 错误场景和调试

#### 常见错误代码
- `-2` (API_EARGS): 无效参数，检查参数传递
- `-9` (API_ENOENT): 资源未找到，通常由错误的tag分配引起
- `-14` (API_EKEY): 密码学错误，检查nodehandle有效性

#### 调试日志
实现包含详细的调试日志：
```cpp
LOG_debug << "Putnodes for deduplication: parent=" << toNodeHandle(f->h) 
          << " filename=" << sname << " nodekey_size=" << tc.nn[0].nodekey.size();
```

### 8. 优势

1. **带宽节省**: 避免重新上传相同文件内容
2. **时间效率**: 重复文件的"上传"完成更快
3. **存储优化**: 服务器端去重减少存储使用
4. **安全性**: MAC验证确保密码学完整性

## 参考图片

- `upload_normal.jpg` - 正常上传流程的日志示例
- `upload_smart.jpg` - 智能去重上传的日志示例

这两张图片展示了实现前后的日志差异，帮助理解去重机制的工作过程。
