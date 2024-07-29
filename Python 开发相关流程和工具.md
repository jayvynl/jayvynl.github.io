# Python 开发相关流程和工具

[TOC]

## Git工作流

[OneFlow](https://www.endoflineblog.com/oneflow-a-git-branching-model-and-workflow) 是一种比 [GitFlow](http://nvie.com/posts/a-successful-git-branching-model/) 更简单的 git 分支模型，One 的含义：只存在一个长期分支，例如 `master` 或 `main`。

### 主分支

主分支命名为 `master` ，是唯一长期分支，和 `GitFlow` 不同，主分支并不一定是稳定分支，而更像 `GitFlow` 中的 `develop` 分支。

### feature 分支

当开发一个新 feature 时，从 `master` 分支切换出 feature 分支：

```bash
$ git checkout -b feature/my-feature master
```


feature 完成后合并到 `master` 分支，使用 rebase 让提交历史保持清晰：

```bash
$ git checkout feature/my-feature
$ git rebase -i master
$ git checkout master
$ git pull origin master
$ git merge --ff-only feature/my-feature
$ git push origin master
$ git branch -d feature/my-feature
```

另一个选择是使用 merge :

```bash
$ git checkout master
$ git pull origin master
$ git merge --no-ff feature/my-feature
$ git push origin master
$ git branch -d feature/my-feature
```

或者 rebase + merge :

```bash
$ git checkout feature/my-feature
$ git rebase -i master
$ git checkout master
$ git pull origin master
$ git merge --no-ff feature/my-feature
$ git push origin master
$ git branch -d feature/my-feature
```

那么到底该使用 rebase 还是 merge 呢？具体使用方式需要根据项目实际情况决定:

- 如果项目组成员数较少，或者代码框架设计良好，不同成员很少会同时修改同一个文件，可以采用 rebase。
- 反之推荐使用 merge，如果 rebase 有冲突，处理不当可能会丢失更改。

### release 分支

当一个迭代的 feature 都开发完成后，从 `master` 分支或某个提交切换出 release 分支：

```bash
$ git checkout -b release/2.3.0 9efc5d
```

基于此 release 分支进行测试、bug fix，待所有缺陷修复完毕，打 tag 并合并回 `mater` 分支：

```bash
$ git checkout release/2.3.0
$ git tag 2.3.0
$ git checkout master
$ git pull origin master
$ git merge release/2.3.0
$ git push --tags origin master
$ git branch -d release/2.3.0
```

### Hotfix 分支

Hotfix 过程类似 release 过程，都会发布一个新版本。

```bash
$ git checkout -b hotfix/2.3.1 2.3.0
```

Hotfix 分支同样需要测试以及 fix，验证通过后，打 tag 并合并回 `mater` 分支：

```bash
$ git checkout hotfix/2.3.1
$ git tag 2.3.1
$ git checkout master
$ git pull origin master
$ git merge hotfix/2.3.1
$ git push --tags origin master
$ git branch -d hotfix/2.3.1
```

如果维护多个版本，且多个版本均存在此缺陷，需要将 hotfix 合并到所有受影响版本，可能需要 cherry pick 。

### 和 GitFlow 区别

1. GitFlow 强制要求 `merge --no-ff` ，只要有合并就会形成一个合并提交。
2. GitFlow 要求所有稳定代码都要合并到主分支（合并两次: `master` `develop`），在主分支上打 tag 发布版本。
3. Hotfix 从主分支开始，修复完成分别合并到 `master` 和 `develop` ，在主分支上打 tag 发布版本。

## python 依赖管理

很多项目直接使用 pip freeze 生成 requirements.txt 文件，然后使用 pip install -r requirements.txt 安装依赖。
这种方式可以在不用长期维护的小项目中使用，但是一般不推荐，原因如下:

1. 依赖升级困难：如果某个 python lib A 存在漏洞，升级 lib A 之后可能会升级一系 lib A 依赖的 lib B，如果 lib C 也依赖 lib B，那么可能导致 lib C 不可用。如果尝试递归升级所有 lib，可能导致用户代码不可用。
2. 依赖移除困难：如果需要移除某个不使用的 lib A，单独移除 lib A 之后可能还有 lib A 依赖的 lib B 残留。

所以我们需要使用 python 依赖管理工具，如 pipenv, poetry, pip-tools 等。下面介绍 pip-tools 的使用。

以下只给出简单示例，更详细使用请参考[官方文档](https://github.com/jazzband/pip-tools)。

### 安装

```shell
pip install pip-tools
```

### 使用

代码中直接 import 的库，手动添加到 requirements.in 文件，然后运行 `pip-compile requirements.in` 生成 requirements.txt 文件。

requirements.in 文件示例：

```
django
requests
```

文件中的依赖都不带版本号，首次执行 `pip-compile` 命令生成的 `requirements.txt` 文件，所有依赖都是最新版本。

除了 requirements.in 之外，还可以增加 requirements-dev.in 存放开发环境才需要的依赖，例如：pytest, coverage, black, flake8 等。
还可以增加 requirements-prod.in 存放生产环境才需要的依赖，例如：gunicorn, uvicorn 等。

requirements-prod.txt 文件示例：

```
-c requirements.txt
gunicorn
```

使用 `pip-compile requirements-prod.txt -o requirements-prod.txt` 生成生产环境的依赖。

如果希望所有依赖始终保持最新，可以在 CI 流程中日常执行 `pip-compile requirements.in --upgrade` 始终保持依赖最新，一旦发现由于升级导致代码不可用，可以及时修复。

### 升级依赖

```shell
# upgrade all packages
$ pip-compile --upgrade

# only update the django package
$ pip-compile --upgrade-package django

# update both the django and requests packages
$ pip-compile --upgrade-package django --upgrade-package requests

# update the django package to the latest, and requests to v2.0.0
$ pip-compile --upgrade-package django --upgrade-package requests==2.0.0

# upgrade all packages whilst constraining requests to the latest version less than 3.0
$ pip-compile --upgrade --upgrade-package 'requests<3.0'
```

### 安装依赖

```shell
$ pip-sync requirements.txt requirements-dev.txt
```

或者

```shell
$ pip install -r requirements.txt -r requirements-dev.txt
```

如果是长期使用的环境，建议使用 pip-sync 命令，此命令除了升级依赖，还会删除不再需要的依赖。

如果是全新的环境，且不希望安装 pip-tools(例如构建 docker 镜像时，希望镜像体积尽可能小)，可以使用 pip install 命令。


## 代码检查工具

代码格式化或者静态检查，可以交给工具来完成。如果是多人协作项目，应该将工具安装、使用、配置文件放在版本控制中，这样其他开发者或者 CI 流程可以一键安装、配置，保证结果可重复。下面以 flake8, black, isort 等工具为例介绍。

### 安装

```shell
$ pip install flake8 black isort
```

可以将 flake8 black isort 放到 requirements-dev.in 中。

### 使用

```shell
$ black .
$ isort .
$ flake8 .
```

3个命令位置参数都支持指定文件或文件夹，限值命令执行对象。更详细的参数就不介绍了，看官方文档。

## pre-commit

[pre-commit](https://pre-commit.com/) 是一个 git hook 工具，可以在执行 `git commit` 时自动执行一些其他命令，例如 flake8, black, isort 等，防止将未检查的代码提交到仓库。

### 安装

```shell
$ pip install pre-commit
```

可以将 pre-commit 放到 requirements-dev.in 中。

### 配置

将上文提到的工具添加到 pre-commit 配置文件中，在项目根目录创建 .pre-commit-config.yaml 文件，内容如下：

```yaml 
repos:
- repo: https://github.com/psf/black
  rev: 22.10.0
  hooks:
  - id: black
- repo: https://github.com/pycqa/isort
  rev: 5.11.2
  hooks:
  - id: isort
- repo: https://github.com/psf/black-pre-commit-mirror
  rev: 24.4.2
  hooks:
  - id: black
- repo: https://github.com/jazzband/pip-tools
  rev: 7.4.1
  hooks:
  - id: pip-compile
    name: pip-compile requirements.in
    args: [requirements.in]
    files: ^requirements\.(in|txt)$
  - id: pip-compile
    name: pip-compile requirements-dev.in
    args: [requirements-dev.in]
    files: ^requirements\-dev\.(in|txt)$
  - id: pip-compile
    name: pip-compile requirements-prod.in
    args: [requirements-prod.in]
    files: ^requirements\-prod\.(in|txt)$
```

### 使用

执行 `pre-commit install` 安装 pre-commit hook。后续这些插件会在执行 `git commit` 时自动执行。

### make

make 实际上是一个构建工具，可以定义一些任务，然后通过 make 命令执行，在编译型语言项目中很常见。但是在 python 项目中，也可以当做命令配置工具来使用。例如我们上文中提到的这些所有工具，可以将所有安装、使用命令放到 makefile 中，通过 make 命令执行，方便使用。

在项目跟目录下创建 Makefile 文件，内容如下：

```makefile
.PHONY: all setup format lint pip-compile

all: format lint pip-compile

setup:
   pip install -r requirements-dev.txt

format:
	black .
	isort .

lint:
	flake8 .

pip-compile:
	pip-compile requirements.in
	pip-compile requirements-dev.in
	pip-compile requirements-prod.in
```

使用：

```shell
make setup
make all
```

## code review

code review 的前提是所有项目协作者都使用同一代码规范，试想如果每个成员的代码风格不一致，那么不同成员提交的代码中可能会掺杂一些代码格式化工具修改的行，干扰真正需要 review 的代码行。良好的代码风格能提高代码的可读性，便于开展 code review。

除了统一风格以外，最好也要有代码检查和单元测试，提前发现不需要人工检查的问题，提高 code review 效率。

code review 的优点大致如下：

- 提早发现代码缺陷，提升代码质量。
- 知道代码会被其他人 review，开发者会更加倾向于写出高质量代码。
- 利于项目成员互相学习，好的库、设计模式、功能实现能够通过 code review 在成员中扩散。

代码托管平台 GitHub、GitLab、Gitee 都支持 Merge Request(GitHub Pull Request)，在 Merge Request 中，可以添加 reviewer，reviewer 可以 review 代码，并给出 review 意见。

为了更好执行 code review：

- 设置主分支为保护分支，禁止直接向主分支提交代码。
- 设置 code review 策略，例如，每个 PR 必须由两个 reviewer approve。
- 控制每个 Merge Request 的代码量，你肯定不想一次 review 2000 行代码。
- 设置邮件或群机器人通知 review。

code review 大体上分为两种：

- 日常 code review，就是上面讲的通过 GitHub Pull Request 的方式。
- code review 会议，当代码开发到一定阶段，由开发者上台讲解自己的设计思路以及关键逻辑的实现，其他成员可以进行提问和讨论。code review 会议可以针对关键代码、设计良好的代码、设计糟糕的代码、抽查代码，定期执行。


## 其他推荐阅读

1. 代码提交规范：[conventional commits](https://www.conventionalcommits.org)。
2. 分支命名规范：https://stackoverflow.com/a/6065944。
3. [跟我一起写Makefile](https://seisman.github.io/how-to-write-makefile/overview.html)。
