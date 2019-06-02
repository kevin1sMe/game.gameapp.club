---
layout: post
title:  "利用dockerignore加速镜像构建"
author: kevin
categories: docker
tags: docker
---
* content
{:toc}

> 目前对于项目中构建镜像较慢的问题，经常让人等的着急。是目录结构安排的不合理，还是手法的不准确？

经常的构建一个镜像在发送Context至daemon花费大量时间，这个过程在官方文档中提到了[dockerignore](https://docs.docker.com/engine/reference/builder/)有说明,
不过比较简要。在构建镜像时，需要向docker引擎发送上下文，比如你执行`docker bulid -t img:tag -f Dockerfile . `时，便会默认将当前目录下的所有文件都发向引擎。
这显然在很多时候都是无必要的，优化空间巨大。

我们知道一个Dockerfile它依赖的文件是可预期的。一般来说所在`ADD`和`COPY`命令的东西，都是构建时需要的。所以优化关键是以白名单模式去代替`dockerignore`的黑名单。默认禁掉所有文件，只有从Dockerfile中提取到的文件才能加入Context。

最终的脚本很简单。 当然为了便利添加了一些小特性。
执行:
>`sh docker_build.sh shortname tag [build or not]`

即可快速在当前目录查找名为Dockerfile-[shortname]的Dockerfile然后构建出名为shortname:tag的镜像。

文末直接附上源码。

```shell
#!/bin/bash
#file: docker_build.sh

# 1. 根据name匹配到合适的dockerfile
# 2. 根据dockerfile计算文件依赖, 生成临时.dockerignore
# 3. 根据tag生成对应的镜像

#Usage: ./docker_build name tag [skip_build|build]
if [ $# -lt 2 ]; then
    echo "Usage: $0 name tag [skip_build|build]"
    exit -1
fi

name=$1
tag=$2
skip_build=$3

echo ${skip_build:="build"} > /dev/null

g_docker_file=


## Error
function error(){
    echo -e "\033[31m\033[01m${cur_date} [error] $1 \033[0m"
}

## warning
function warn(){
    echo -e "\033[33m\033[01m${cur_date} [warn] $1 \033[0m"
}

## info
function info(){
    echo -e "\033[32m\033[01m${cur_date} [info] $1 \033[0m"
}


function find_dockerfile()
{
    result_cnt=`find . -maxdepth 1 -type f -name "Dockerfile*$1*" | wc -l`
    if [ $result_cnt -gt 1 ]; then
        warn "Find many dockerfile match with [Dockerfile*$1*]"
        return 1
    elif [ $result_cnt -eq 0 ]; then
        error "Not find any dockerfile match with [Dockerfile*$1*]"
        return 2
    fi
    result=`find . -maxdepth 1 -type f -name "Dockerfile*$1*"`
    g_docker_file=$result
    return 0
}

function generate_docker_ignore()
{
    docker_file=$1
    ignore_file=$2

    :> $ignore_file
    echo "*" >> $ignore_file
    echo "!${ignore_file}" >> $ignore_file
    echo "!${g_docker_file}" >> $ignore_file
    cat $g_docker_file | egrep "^COPY|^ADD"  | awk '{printf("!%s\n", $2)}' >> $ignore_file

    sed  -i 's/^!\.\//!/' $ignore_file
}


# 1.根据name匹配到合适的dockerfile
find_dockerfile $name
if [ $? -ne 0 ]; then
    exit -1
fi
info "================ $g_docker_file ==========="
cat $g_docker_file


ignore_file=.dockerignore
# 2. 根据dockerfile计算文件依赖, 生成临时.dockerignore
info "================ .dockerignore ==========="
generate_docker_ignore $g_docker_file $ignore_file
cat $ignore_file
info "==========================================="

# 3. 根据tag生成对应的镜像
if [ $skip_build == "build" ]; then
    docker build -t $name:$tag -f $g_docker_file .
    if [ $? -eq 0 ]; then
        info "build image succ!"
    else
        warn "build image failed"
    fi
else
    warn "skip build image!"
fi
```
