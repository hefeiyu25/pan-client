# Pan Client

一个统一的 Go 语言云盘客户端库，支持多种云存储服务，提供一致的 API 接口。

[![Go Version](https://img.shields.io/badge/Go-1.23+-blue.svg)](https://golang.org)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

## ✨ 特性

- **多云盘支持** - 统一接口操作多种云存储服务
- **断点续传** - 支持上传和下载的断点续传
- **多线程传输** - 并发上传下载提升性能
- **目录递归** - 支持目录的递归上传和下载
- **文件过滤** - 按扩展名、文件名进行过滤
- **进度回调** - 实时获取传输进度
- **配置持久化** - 自动保存配置和缓存
- **错误重试** - 内置错误处理和重试机制

## 🚀 支持的服务

| 云盘服务 | 驱动类型 | 状态 |
|---------|---------|------|
| Cloudreve | `cloudreve` | ✅ 完整支持 |
| 夸克云盘 | `quark` | ✅ 完整支持 |
| 迅雷浏览器 | `thunder_browser` | ✅ 完整支持 |

## 📦 安装

```bash
go get github.com/hefeiyu2025/pan-client
```

## 🔧 快速开始

### 1. 基本使用

```go
package main

import (
    "fmt"
    "github.com/hefeiyu2025/pan-client"
    _ "github.com/hefeiyu2025/pan-client/pan/driver" // 导入所有驱动
)

func main() {
    // 获取 Cloudreve 客户端
    client, err := pan.GetClient(pan.Cloudreve)
    if err != nil {
        panic(err)
    }
    
    // 获取磁盘信息
    disk, err := client.Disk()
    if err != nil {
        panic(err)
    }
    fmt.Printf("总空间: %d MB, 已用: %d MB, 剩余: %d MB\n", 
        disk.Total, disk.Used, disk.Free)
}
```

### 2. 上传文件/目录

```go
// 上传单个文件
err = client.UploadFile(pan.UploadFileReq{
    LocalFile:  "./local/file.pdf",
    RemotePath: "/remote/folder",
    Resumable:  true, // 启用断点续传
})

// 上传整个目录
err = client.UploadPath(pan.UploadPathReq{
    LocalPath:   "./local/data",
    RemotePath:  "/backup",
    Resumable:   true,
    Extensions:  []string{".pdf", ".doc"}, // 只上传指定类型
    IgnorePaths: []string{"temp"},         // 忽略目录
    SuccessDel:  false,                    // 上传成功后删除本地文件
})
```

### 3. 下载文件/目录

```go
// 下载单个文件
err = client.DownloadFile(pan.DownloadFileReq{
    RemoteFile:  fileObj,          // 从 List 获取的 PanObj
    LocalPath:   "./downloads",
    Concurrency: 4,                // 4线程并发
    ChunkSize:   50 * 1024 * 1024, // 50MB 分块
    OverCover:   true,             // 覆盖已存在文件
})

// 下载整个目录
err = client.DownloadPath(pan.DownloadPathReq{
    RemotePath:  &pan.PanObj{Path: "/", Name: "backup"},
    LocalPath:   "./local/backup",
    Concurrency: 4,
    Extensions:  []string{".pdf"},
    NotTraverse: false, // 是否遍历子目录
})
```

### 4. 文件操作

```go
// 列出目录
objs, err := client.List(pan.ListReq{
    Dir: &pan.PanObj{Path: "/", Name: "documents"},
    Reload: true, // 强制刷新缓存
})

// 创建目录
newDir, err := client.Mkdir(pan.MkdirReq{
    NewPath: "/backup/2024",
})

// 重命名
err = client.ObjRename(pan.ObjRenameReq{
    Obj:     fileObj,
    NewName: "new_name.pdf",
})

// 批量重命名
err = client.BatchRename(pan.BatchRenameReq{
    Path: dirObj,
    Func: func(obj *pan.PanObj) string {
        return fmt.Sprintf("prefix_%s", obj.Name)
    },
})

// 移动文件
err = client.Move(pan.MovieReq{
    Items:     []*pan.PanObj{file1, file2},
    TargetObj: targetDir,
})

// 删除文件
err = client.Delete(pan.DeleteReq{
    Items: []*pan.PanObj{file1, file2},
})
```

### 5. 高级功能

```go
// 获取直链
links, err := client.DirectLink(pan.DirectLinkReq{
    List: []*pan.DirectLink{
        {FileId: "123", Name: "file.pdf"},
    },
})

// 设置自定义配置读写
client, err = pan.GetClientByRw(
    "custom-id",
    pan.Cloudreve,
    func(config pan.Properties) error {
        // 自定义读取配置
        return nil
    },
    func(config pan.Properties) error {
        // 自定义写入配置
        return nil
    },
)
```

## ⚙️ 配置文件

配置文件 `pan-client.yaml` 示例：

```yaml
driver:
  cloudreve:
    url: https://pan.example.com
    session: your_session_cookie
    type: hucl
    chunk_size: 104857600  # 100MB
    skip_verify: false
    refresh_time: 0
    
  quark:
    id: your_quark_id
    pus: your_pus_token
    puus: your_puus_token
    chunk_size: 104857600
    
  thunder_browser:
    access_token: your_token
    refresh_token: your_refresh_token
    username: your_username
    password: your_password
    device_id: your_device_id

log:
  enable: true
  file_name: app.log
  max_size: 50      # MB
  max_backups: 30
  max_age: 28       # days
  compress: false

server:
  cache_file: cache.dat
  debug: true
  download_max_retry: 2
  download_max_thread: 5
  download_tmp_path: ./tmp
```

## 🔍 核心接口

### Driver 接口

所有云盘驱动都实现以下接口：

```go
type Driver interface {
    Meta      // 元数据操作
    Operate   // 文件操作
    Share     // 分享功能
}
```

### Meta 接口

```go
type Meta interface {
    GetId() string
    Init() (string, error)
    InitByCustom(id string, read, write ConfigRW) (string, error)
    Drop() error
    ReadConfig() error
    WriteConfig() error
    Get(key string) (interface{}, bool)
    Set(key string, value interface{})
    Del(key string)
}
```

### Operate 接口

```go
type Operate interface {
    Disk() (*DiskResp, error)
    List(req ListReq) ([]*PanObj, error)
    ObjRename(req ObjRenameReq) error
    BatchRename(req BatchRenameReq) error
    Mkdir(req MkdirReq) (*PanObj, error)
    Move(req MovieReq) error
    Delete(req DeleteReq) error
    UploadPath(req UploadPathReq) error
    UploadFile(req UploadFileReq) error
    DownloadPath(req DownloadPathReq) error
    DownloadFile(req DownloadFileReq) error
    OfflineDownload(req OfflineDownloadReq) (*Task, error)
    TaskList(req TaskListReq) ([]*Task, error)
    DirectLink(req DirectLinkReq) ([]*DirectLink, error)
}
```

## 📊 数据结构

### PanObj - 云盘对象

```go
type PanObj struct {
    Id     string    // 对象ID
    Name   string    // 名称
    Path   string    // 路径
    Size   int64     // 大小（字节）
    Type   string    // 类型：file/dir
    Ext    Json      // 扩展数据
    Parent *PanObj   // 父对象
}
```

### DiskResp - 磁盘信息

```go
type DiskResp struct {
    Used  int64  // 已用空间（MB）
    Free  int64  // 剩余空间（MB）
    Total int64  // 总空间（MB）
}
```

## 🛠️ 开发指南

### 添加新驱动

1. 在 `pan/driver/` 下创建新包
2. 实现 `Driver` 接口
3. 在 `init()` 中注册驱动：

```go
func init() {
    pan.RegisterDriver(pan.NewDriverType, func() pan.Driver {
        return &NewDriver{
            PropertiesOperate: pan.PropertiesOperate[*NewDriverProperties]{
                DriverType: pan.NewDriverType,
            },
            CacheOperate:  pan.CacheOperate{DriverType: pan.NewDriverType},
            CommonOperate: pan.CommonOperate{},
        }
    })
}
```

### 测试

```bash
# 运行测试
go test -v

# 运行特定测试
go test -v -run TestDownloadAndUpload
```

## 📝 使用示例

完整示例请参考 `enter_test.go` 文件。

```go
func TestDownloadAndUpload(t *testing.T) {
    defer GracefulExist()
    
    client, err := GetClient(pan.Cloudreve)
    if err != nil {
        t.Error(err)
        return
    }
    
    // 上传
    err = client.UploadPath(pan.UploadPathReq{
        LocalPath:  "./tmpdata",
        RemotePath: "/test1",
        Resumable:  true,
        Extensions: []string{".pdf"},
    })
    
    // 列出并下载
    list, err := client.List(pan.ListReq{
        Dir:    &pan.PanObj{Path: "/", Name: "test1"},
        Reload: true,
    })
    
    for _, item := range list {
        if item.Type == "file" {
            err = client.DownloadFile(pan.DownloadFileReq{
                RemoteFile: item,
                LocalPath:  "./tmpdata",
                Concurrency: 2,
            })
        }
    }
}
```

## 🔒 安全说明

- 所有敏感信息（token、session）都应存储在配置文件中
- 建议将配置文件添加到 `.gitignore`
- 使用环境变量或密钥管理服务存储生产环境凭证

## 🤝 贡献指南

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启 Pull Request

## 📄 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](LICENSE) 文件了解详情

## 🐛 问题反馈

- 提交 Issue
- 提供详细的错误信息和复现步骤
- 附上相关日志和配置

## 🙏 致谢

- [Go](https://golang.org/) - 优秀的编程语言
- [logrus](https://github.com/sirupsen/logrus) - 日志库
- [viper](https://github.com/spf13/viper) - 配置管理
- [req](https://github.com/imroc/req) - HTTP 客户端

---

**Pan Client** © 2025 - Made with ❤️ by hefeiyu2025