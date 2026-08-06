
## 1、安装

```
npm install -g @fission-ai/openspec@latest
```


## 2、使用

参考：`https://github.com/Fission-AI/OpenSpec/blob/main/docs/getting-started.md`

安装成功后可以创建一个项目，然后进入到项目目录中
```
cd your-project
openspec init
```

这里就进行了初始化，会让你选择一些工具，比如 claude code 或者 codebuddy 等等，使用空格选中，然后回车。

您的项目结构如下：

```
openspec/
├── specs/              # Source of truth (your system's behavior)
│   └── <domain>/
│       └── spec.md
├── changes/            # Proposed updates (one folder per change)
│   └── <change-name>/
│       ├── proposal.md
│       ├── design.md
│       ├── tasks.md
│       └── specs/      # Delta specs (what's changing)
│           └── <domain>/
│               └── spec.md
└── config.yaml         # Project configuration (optional)
```

**两个关键目录：**

- **`specs/`**- 真理的来源。这些规范描述了您的系统当前的行为方式。按领域组织（例如`specs/auth/`，，`specs/payments/`）。
    
- **`changes/`**- 建议的修改。每项修改都会生成一个单独的文件夹，其中包含所有相关的文件。修改完成后，其规范将合并到主`specs/`目录中。

这里会创建一些基本的命令和 skill。

**默认快速路径（核心配置文件）：**

```
/opsx:propose ──► /opsx:apply ──► /opsx:archive
```

**展开路径（自定义工作流程选择）：**

```
/opsx:new ──► /opsx:ff or /opsx:continue ──► /opsx:apply ──► /opsx:verify ──► /opsx:archive
```

一般只会创建一些默认的命令路径，如果想扩展，可以使用 `openspec config` 进行扩展。

然后不管是新项目还是老项目，我们先要对项目进行一些初始化的介绍，比如项目背景，技术架构，完善 `openspec/config.yaml`。

同时还可以事先让 openspec 帮忙创建一些 skill 或者从外部拷贝一些 skill 过来（可以让 openspec 创建 skill 对应的命令）。


## 3、相关需求开发流程











