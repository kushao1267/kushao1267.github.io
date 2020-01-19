---
layout:     post
title:      "Poetry调研与使用"
date:       2020-01-17 17:00:43
author:     "Jonliu"
header-img: "img/post-bg-2018.jpg"
tags:
    - Python
---

# Poetry调研

## 一、对比

| | Poetry | Pipenv |
| --- | --- | --- |
| 对比版本 | 1.0.2 | 2018-11-26 |
| 依赖文件名 | Pipfile, Pipefile.lock | pyproject.toml, poetry.lock |
| 依赖文件语法 | TOML | TOML |
| Python版本管理 | ✓ | ✓ |
| requirements.txt导入 | ✗ | ✓ |
| requirements.txt导出 | ✓ | ✓ |
| 依赖锁定 | ✓，Lock较快 | ✓，Lock慢，且有lock cache的bug，需要--clear |
| 包发布功能(代替setup.py) | ✓ | ✗ |
| 平台支持| Macos,linux,windows支持良好 | Macos,linux支持良好,Windows支持较差 |
| Stars(2020-01-17) | 8.7k | 19.4k |
| 近期活跃度 | 较高 | 低 |
| 影响力| 无Python核心开发者 | 有Python核心开发者，PyPA推荐 |
| 关键痛点 | 对虚拟环境的管理控制有些弱，没有 Pipenv 那样的删除虚拟环境和清空依赖的操作。 | 安装某个依赖默认会更新不相关的且已经锁定的依赖，需要额外指定--keep-outdated参数。 |

综合[李辉](https://www.zhihu.com/people/im-greyli/activities)和[董伟明](https://www.zhihu.com/people/dongweiming/activities)对Python包管理工具上的讨论，达成共识的是：可用于生产环境且最稳定的方式还是：virtualenv + pip

## 二、使用
### 安装方式
安装(有墙): `$ curl -sSL https://raw.githubusercontent.com/sdispater/poetry/master/get-poetry.py | python3`
配置环境变量: `$ echo 'export PATH=$PATH:$HOME/.poetry/bin' >> ~/.zshrc`
确认安装成功: 运行
`$ poetry --version
Poetry version 1.0.2
`

### Python环境
查看当前Python环境: `$ poetry env info -p`
设置Python环境: `$ poetry env use 3.6`

### 初始化poetry仓库
新建项目则:
`$ poetry new {proj_name}`
如果已经有项目，则在项目路径下执行:
`$ poetry init`
执行完成后，项目路径下会有pyproject.toml文件，该文件保存项目的依赖，替代了requirements.txt和Pipfile文件。

### 创建虚拟环境
首先，保证项目路径下有pyproject.toml文件，然后执行
`$ poetry install`，这个命令会读取 pyproject.toml 中的所有依赖（包括开发依赖）并安装，如果不想安装开发依赖，可以附加 --no-dev 选项。如果项目根目录有 poetry.lock 文件，会安装这个文件中列出的锁定版本的依赖。

### 激活虚拟环境
快速在当前目录对应的虚拟环境中执行命令:
`$ poetry run python app.py`
显式地激活虚拟环境:
`$ poetry shell`

### 添加/删除依赖
`$ poetry add {dep_name}[==dep_version] [--dev]`
`$ poetry remove {dep_name}`

### 查看依赖关系
`$ poetry show --tree`

### 锁定依赖
`$ poetry lock`
将会自动生成poetry.lock文件。

### 导出依赖
`$ poetry export --without-hashes --format=requirements.txt -o requirements_new.txt`

### 删除虚拟环境
进入虚拟环境shell，查看当前的python版本，即可知道虚拟环境的路径，退出当前虚拟环境的shell。
`$ poetry shell
$ which python
$ exit`
最后，删除相关shell。一般Macos的路径是:`~/Library/Caches/pypoetry/virtualenvs`

## 相关参考:
[1.不要用Pipenv](http://greyli.com/do-not-use-pipenv/)
[2.相比 Pipenv，Poetry 是一个更好的选择](http://greyli.com/poetry-a-better-choice-than-pipenv/)
[3.也谈「不要用 Pipenv」](https://zhuanlan.zhihu.com/p/80683249)
[4.Frequently Encountered Pipenv Problems](https://pipenv-fork.readthedocs.io/en/latest/diagnose.html)
[5.要不我们还是用回 virtualenv/venv 和 pip 吧](http://greyli.com/back-to-virtualenv-venv-and-pip/)
[6.Pipenv and Poetry: Benchmarks & Ergonomics](https://johnfraney.ca/posts/2019/03/06/pipenv-poetry-benchmarks-ergonomics/)


