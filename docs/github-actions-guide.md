# GitHub Actions 教程

这份文档面向日常项目开发，重点放在怎么写、怎么跑、怎么排错，而不是罗列全部语法。

## 1. GitHub Actions 是什么

GitHub Actions 是 GitHub 提供的 CI/CD 平台，可以在代码发生变化时自动执行任务，比如：

- 安装依赖
- 运行测试
- 检查代码格式
- 构建产物
- 自动发布

工作流文件通常放在 `.github/workflows/` 目录下，使用 YAML 编写。

## 2. 核心概念

一个 Actions 工作流通常由下面几个部分组成：

- `workflow`：整个工作流文件
- `on`：触发条件
- `jobs`：一个或多个任务
- `steps`：任务中的执行步骤
- `runner`：执行环境，比如 `ubuntu-latest`
- `action`：别人封装好的可复用步骤

最小结构如下：

```yml
name: CI

on:
  push:
    branches: ["master"]
  pull_request:
    branches: ["master"]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run script
        run: echo "Hello GitHub Actions"
```

## 3. 常见目录结构

```text
.github/
  workflows/
    ci.yml
    release.yml
```

建议按用途拆文件：

- `ci.yml`：提交和 PR 检查
- `release.yml`：发版流程
- `deploy.yml`：部署流程

## 4. 常用触发方式

### `push`

在代码推送时触发：

```yml
on:
  push:
    branches: ["master", "main"]
```

### `pull_request`

在提交 PR 时触发：

```yml
on:
  pull_request:
    branches: ["master", "main"]
```

### `workflow_dispatch`

手动触发：

```yml
on:
  workflow_dispatch:
```

### 多个触发器一起使用

```yml
on:
  push:
    branches: ["master"]
  pull_request:
    branches: ["master"]
  workflow_dispatch:
```

## 5. Node.js 项目常用工作流

这是最常见的一类写法：拉代码、装 Node、装依赖、跑测试。

```yml
name: Node CI

on:
  push:
    branches: ["master"]
  pull_request:
    branches: ["master"]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run lint
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Build project
        run: npm run build
```

## 6. 常用 Action

### `actions/checkout`

把仓库代码拉到 runner 上。

```yml
- uses: actions/checkout@v4
```

### `actions/setup-node`

设置 Node.js 版本。

```yml
- uses: actions/setup-node@v4
  with:
    node-version: 20
```

### `actions/cache`

缓存依赖或构建结果，加快执行速度。

```yml
- name: Cache npm
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

说明：

- `path`：要缓存的路径
- `key`：缓存命中键
- `restore-keys`：模糊匹配回退键

### `actions/upload-artifact`

上传构建产物。

```yml
- name: Upload build output
  uses: actions/upload-artifact@v4
  with:
    name: dist
    path: dist
```

### `actions/download-artifact`

在后续 job 下载产物。

```yml
- name: Download build output
  uses: actions/download-artifact@v4
  with:
    name: dist
```

## 7. 常用命令和写法

这里的“命令”通常指 `run` 里执行的 shell 命令，以及 Actions 表达式写法。

### 执行单行命令

```yml
- name: Print current directory
  run: pwd
```

### 执行多行命令

```yml
- name: Run multiple commands
  run: |
    node -v
    npm -v
    npm ci
    npm test
```

### 设置环境变量

```yml
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      NODE_ENV: production
    steps:
      - run: echo "$NODE_ENV"
```

也可以给单个步骤设置：

```yml
- name: Use custom env
  env:
    API_URL: https://example.com
  run: echo "$API_URL"
```

### 使用 secrets

在仓库 `Settings > Secrets and variables > Actions` 中配置。

```yml
- name: Use token
  env:
    MY_TOKEN: ${{ secrets.MY_TOKEN }}
  run: echo "token is configured"
```

不要直接输出 secret 内容。

### 使用表达式

```yml
if: github.ref == 'refs/heads/master'
```

```yml
if: success()
```

```yml
if: failure()
```

### 条件执行步骤

```yml
- name: Run only on master
  if: github.ref == 'refs/heads/master'
  run: echo "deploy on master"
```

### 指定工作目录

```yml
- name: Install frontend deps
  working-directory: frontend
  run: npm ci
```

### 指定 shell

```yml
- name: Use bash
  shell: bash
  run: echo "hello"
```

## 8. 矩阵构建

如果你想在多个 Node 版本或多个系统上跑同一套流程，可以用矩阵。

```yml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - run: npm ci
      - run: npm test
```

优点：

- 一份配置覆盖多个版本
- 适合验证兼容性

缺点：

- 执行时间和资源消耗更高

## 9. 多 Job 工作流

可以把构建和部署拆开：

```yml
name: Build And Deploy

on:
  push:
    branches: ["master"]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
      - run: echo "deploy here"
```

说明：

- `needs: build` 表示 `deploy` 依赖 `build`
- 前一个 job 成功后，后一个 job 才会开始

## 10. 常见最佳实践

### 1. 固定 Action 版本

推荐：

```yml
uses: actions/checkout@v4
```

不推荐：

```yml
uses: actions/checkout@main
```

原因：

- 避免上游变更导致工作流不稳定
- 便于排查问题

### 2. 使用 `npm ci` 而不是 `npm install`

CI 场景推荐：

```yml
run: npm ci
```

原因：

- 更快
- 更稳定
- 严格依赖锁文件

### 3. 拆分职责清晰的 Job

推荐把检查流程拆成：

- `lint`
- `test`
- `build`
- `deploy`

好处：

- 失败点更清楚
- 更容易并行执行
- 更方便复用和维护

### 4. 开启依赖缓存

对于 npm、pnpm、yarn，尽量使用缓存，减少重复下载。

### 5. 不要在日志里打印敏感信息

包括：

- token
- 密钥
- 私有地址
- 生产环境配置

### 6. 尽量在 PR 阶段就跑检查

推荐同时监听：

- `push`
- `pull_request`

这样可以尽早拦住问题，而不是等合并后再失败。

### 7. 用明确的步骤名

推荐：

```yml
- name: Install dependencies
- name: Run unit tests
- name: Build production bundle
```

不推荐：

```yml
- name: test
- name: run
```

步骤名清楚，排错效率会高很多。

### 8. 尽量减少重复逻辑

如果多个工作流重复安装、测试、构建逻辑，可以：

- 抽成复用工作流
- 抽成脚本文件
- 在 `package.json` 里统一命令入口

### 9. 部署前增加保护条件

例如只允许 `master` 分支部署：

```yml
if: github.ref == 'refs/heads/master'
```

### 10. 保持工作流小而稳定

不要把所有事情都堆到一个文件里。职责单一的工作流更好维护。

## 11. 常见错误

### 1. YAML 缩进错误

YAML 对缩进非常敏感，必须统一空格，不能随意混用。

错误示例：

```yml
steps:
 - uses: actions/checkout@v4
```

推荐统一两空格缩进。

### 2. `with` 写错位置

错误示例：

```yml
- uses: actions/checkout@v4
  with:
    node-version: 20
```

原因：

- `node-version` 是 `actions/setup-node` 的参数
- 不是 `actions/checkout` 的参数

正确示例：

```yml
- uses: actions/checkout@v4

- uses: actions/setup-node@v4
  with:
    node-version: 20
```

### 3. 冒号后面没有空格

错误示例：

```yml
node-version:20
```

正确示例：

```yml
node-version: 20
```

这类问题会导致 GitHub 报 YAML 或 workflow schema 错误。

### 4. 分支名写错

如果你的默认分支是 `main`，但工作流里写的是 `master`，那工作流可能根本不会触发。

### 5. 本地命令能跑，Actions 里失败

常见原因：

- Node 版本不一致
- 缺少环境变量
- 系统环境差异
- 依赖未锁定
- 文件路径大小写问题

### 6. `npm install` 和锁文件不一致

如果项目提交了 `package-lock.json`，但依赖树已经漂移，CI 里可能失败。优先修锁文件，而不是在 CI 里绕过。

## 12. 排错建议

### 先看哪一步失败

GitHub Actions 日志会明确标出失败的 job 和 step，先确认是：

- checkout 失败
- 安装依赖失败
- 测试失败
- 构建失败
- 权限失败

### 先看完整报错，不要只看最后一行

很多时候真正原因在上面几行，例如：

- 找不到命令
- 版本不兼容
- 权限不足
- 文件不存在

### 在必要时打印调试信息

```yml
- name: Debug environment
  run: |
    pwd
    ls -la
    node -v
    npm -v
```

### 检查当前分支和事件

一些条件判断依赖：

- `github.ref`
- `github.event_name`
- `github.actor`

必要时可以打印：

```yml
- name: Debug github context
  run: |
    echo "${{ github.ref }}"
    echo "${{ github.event_name }}"
```

## 13. 一个更实用的 CI 模板

```yml
name: CI

on:
  push:
    branches: ["master"]
  pull_request:
    branches: ["master"]
  workflow_dispatch:

jobs:
  lint-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint --if-present

      - name: Test
        run: npm test --if-present

      - name: Build
        run: npm run build --if-present
```

适合大多数前端或 Node 项目起步使用。

## 14. 建议的落地顺序

如果你要给项目补 Actions，建议按这个顺序来：

1. 先写最小 CI：`checkout + setup-node + npm ci + test`
2. 确认在 PR 上能稳定运行
3. 再加 lint 和 build
4. 再加缓存
5. 最后才加部署和发布

这样更容易定位问题，也不容易把工作流一次写得过重。

## 15. 总结

写 GitHub Actions 时，优先关注三件事：

- 触发条件对不对
- Action 用法对不对
- 命令是否能在干净环境里稳定执行

最常见的问题不是语法本身，而是：

- 参数写到了错误的 action 上
- 分支名写错
- 本地环境和 CI 环境不一致
- secrets、缓存、锁文件没有处理好

如果只是刚开始用，先从一个能稳定跑通的最小工作流开始，再逐步增加能力。
