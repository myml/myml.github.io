---
title: obs 添加玲珑构建支持
date: 2023-04-07
draft: false
tags: ["玲珑", "deepin"]
categories: ["工具"]
---

[obs](https://openbuildservice.org/) 全称 Open Build Service，是一个开放的构建平台。相较于其他构建工具有以下优点：

- 支持跨平台构建（x86、arm64 等）
- 支持多种虚拟环境（kvm、lxc、chroot 等）
- 支持软件包构建（deb、rpm、pkg 等）
- 支持容器构建（flatpak、appimage、docker 等）
- 支持发型版镜像构建（debian、windows 等）

[玲珑](https://linglong.dev/) 是一种新型的独立包管理工具集，致力于治理 Linux 系统下传统软件包格式复杂、交叉的依赖关系导致的各种兼容性问题，以及过于松散的权限管控导致的安全风险。

本文介绍怎么给 obs 添加玲珑构建支持，供以参考实现你自己的 obs 构建服务。

<!--more-->

其他一些模板化的改动就不再赘述，可以从 [pr1](https://github.com/linuxdeepin/obs-build/pull/1) 和 [pr2](https://github.com/linuxdeepin/open-build-service/pull/1) 查看，本文只介绍关键的几个部分。

## 构建文件解析

构建自定义的包格式，需要让 obs 读懂构建配置文件，玲珑的配置文件是 yaml 格式，里面包含应用的 id、版本、依赖、运行时等信息，需要编写 perl 模块用于解析 linglong.yaml 文件

在 obs-build 项目的 Build 目录创建 Linglong.pm，代码如下

```perl
# 包名
package Build::Linglong;

use strict;

eval { require YAML::XS; $YAML::XS::LoadBlessed = 0; };
*YAML::XS::LoadFile = sub {die("YAML::XS is not available\n")} unless defined &YAML::XS::LoadFile;

# 玲珑包的依赖规则转换成 obs 的依赖规则，需要注意处理范围依赖等情况
sub toDepth {
  my($package) = @_;
  my $id = $package->{'id'};
  my $version = $package->{'version'};
  # 判断是否将包名和版本号写一起的简写模式，如 org.deepin.base/20.5.12
  my @spl = split('/', $package->{'id'});
  if(scalar(@spl)>1) {
    $id = $spl[0];
    $version = $spl[1];
  }
  # debian 包名不支持大写，转为小写字符
  $id = lc($id);
  # 如果版本号小余四位，转为范围限制 
  # 如 org.deepin.base/20.5.12 会转为 linglong.org.deepin.base (>= 20.5.12), linglong.org.deepin.base (< 20.5.13)
  my @vs = split('\.', $version);
  if(scalar(@vs) < 4) {
    my $min = $version;
    $vs[-1] = $vs[-1]+1;
    my $max = join('.', @vs);
    return 'linglong.'.$id.' (>= '.$min.'), '.'linglong.'.$id.' (<< '.$max.')'
  }
  # 版本号是四位，则使用固定限制
  return 'linglong.'.$id.' (= '.$version.')'
}
# 解析构建配置的函数，在外部使用
sub parse {
  my ($cf, $fn) = @_;

  my $yml;
  eval { $yml = YAML::XS::LoadFile($fn); };
  return {'error' => "Failed to parse yml file"} unless $yml;
  # ret 是函数返回值，需要包含几个必选项
  my $ret = {};
  # name 和 version 是必选项，name 就是包名，在玲珑里是 id
  $ret->{'name'} = $yml->{'package'}->{'id'};
  $ret->{'version'} = $yml->{'package'}->{'version'} || "0";

  # 玲珑应用有 base,runtime,depends 三种依赖，全部通过 $ret->{'deps'} 返回
  my @packdeps;
  if($yml->{'runtime'}) {
    push @packdeps, toDepth($yml->{'runtime'}); 
  }

  if($yml->{'base'}) {
    push @packdeps, toDepth($yml->{'base'});
  }

  if($yml->{'depends'}) {
    for my $depend (@{$yml->{'depends'}}) {
      push @packdeps, toDepth($depend);
    }
  }

  $ret->{'deps'} = \@packdeps;
  # ret 源码，玲玲构建是暂时忽略这个字段
  my @sources;
  $ret->{'sources'} = \@sources;

  return $ret;
}

1;
```

## 依赖仓库封装

解析出构建文件后，obs 会从仓库查询相关依赖包是否可用，并且会在构建之前进行下载和安装，而 obs 支持的仓库主要是 rpm 和 debian 等常见仓库。

玲珑或者你自定义的包格式仓库则不受支持，如果要添加仓库支持，需要较大的工作量。
玲珑这里偷了个懒，将玲珑包进行了二次封装，打包成 deb 格式，通过 deb 的构子脚本，在安装 deb 的同时安装玲珑包。

打包 deb 需要生成一些模板文件，我这里是编写了 obs 的 [Source Services](https://openbuildservice.org/help/manuals/obs-user-guide/cha.obs.source_service.html) 服务，这个机制主要用于对源码进行透明的预处理，例如官方的 [Tar SCM Service](https://github.com/openSUSE/obs-service-tar_scm) ，可以根据参数从 git、snv 拉取源代码，就不需要手动上传源码了。而我编写的 [Linglong Service](https://github.com/myml/obs-service-linglon) 会生成 deb 所需的 control 和构子脚本，具体代码如下：

```python
#!/usr/bin/python3
import sys
import os
import yaml

# 获取 outdir 参数
# 参数 linglong --file linglong.yaml --outdir $dir
outdir = sys.argv[4]
linglongFile = sys.argv[2]

# 读取 linglong 文件
with open(linglongFile, 'r') as f:
  data = f.read()

# 解析 linglong 文件并拉取 git 源码
lconf = yaml.load(data)
if (lconf['source']['kind']=="git"):
    package = lconf['package']['id'] 
    url = lconf['source']['url']
    source = ".linglong-target/" + package + "/source"
    # git clone $url $source
    os.system("git clone " + url + " " + source)

    # tar cvfz $outdir/git_source.tar.zst .linglong-target
    os.system("tar cvfz " + outdir + "/git_source.tar.zst .linglong-target")

#### 准备打包 deb 所需要的文件 ####
package = lconf['package']['id']
version = lconf['package']['version']
depends = ""

# 转换玲珑依赖为 deb 依赖
def toDebDepend(depend):
    id = depend['id']
    fields = id.split("/")
    if len(fields) > 1:
        id = fields[0]
        version = fields[1]
    else:
        version = str(depend['version'])
    id = id.lower()
    vs = version.split(".")
    if len(vs) <= 3:
        min = ".".join(vs)
        vs[-1]=str(int(vs[-1])+1)
        max = ".".join(vs)
        return f"linglong.{id} (>= {min}), linglong.{id} (<< {max})"
    return f"linglong.{id} (= {version})"

# 生成 control 文件
deps=[]
if "runtime" in lconf:
    deps.append(toDebDepend(lconf['runtime']))
if "base" in lconf:
    deps.append(toDebDepend(lconf['base']))
if "depends" in lconf:
    for i in range(0, len(lconf['depends'])):
        deps.append(toDebDepend(lconf['depends'][i]))
if len(deps)>0:
    depends = "\nDepends: " + ", ".join(deps)
control=f'''Package: linglong.{package.lower()}
Version: {version}
Architecture: amd64
Maintainer: wurongjie <wurongjie@deepin.org>{depends}
Section: linglong
Priority: standard
Description: linglong to deb
'''
with open(outdir + "/control", 'w', encoding='utf-8') as w_f:
    w_f.write(control)

# 生成 postinst 文件
postinst=f'''## Generated by linglong service
repoDir=${{LINGLONG_REPO_DIR:=/var/cache/linglong/repo}}
debDir=${{LINGLONG_DEB_DIR:=/var/cache/linglong/deb}}
mkdir -p $repoDir
ostree init --repo=$repoDir --mode=bare-user-only
ostree --repo=$repoDir commit --branch=linglong/{package}/{version}/x86_64/devel --tree=tar=$debDir/linglong-{package}-{version}-x86_64-devel.tar.zst
ostree --repo=$repoDir commit --branch=linglong/{package}/{version}/x86_64/runtime --tree=tar=$debDir/linglong-{package}-{version}-x86_64-runtime.tar.zst
'''
with open(outdir + "/postinst", 'w', encoding='utf-8') as w_f:
    w_f.write(postinst)

# 生成 postrm 文件
postinst=f'''## Generated by linglong service
repoDir=${{LINGLONG_REPO_DIR:=/var/cache/linglong/repo}}
ostree --repo=$repoDir refs --delete linglong/{package}/{version}/x86_64/devel
ostree --repo=$repoDir refs --delete linglong/{package}/{version}/x86_64/runtime
'''
with open(outdir + "/postrm", 'w', encoding='utf-8') as w_f:
    w_f.write(postinst)

# deb 打包脚本
deb=f'''
#!/bin/bash -e
set -v on
debDir=/var/cache/linglong/deb
mkdir -p DEB/DEBIAN DEB$debDir
cp control postinst postrm DEB/DEBIAN/
chmod +x DEB/DEBIAN/postinst DEB/DEBIAN/postrm
ostree --repo=/home/abuild/.cache/linglong-builder/repo export {package}/{version}/x86_64/devel | zstd -o DEB$debDir/linglong-{package}-{version}-x86_64-devel.tar.zst
ostree --repo=/home/abuild/.cache/linglong-builder/repo export {package}/{version}/x86_64/runtime | zstd -o DEB$debDir/linglong-{package}-{version}-x86_64-runtime.tar.zst
dpkg-deb -b -Znone --root-owner-group ./DEB ../OTHER
'''
with open(outdir + "/deb.bash", 'w', encoding='utf-8') as w_f:
    w_f.write(deb)
```

可以看到这是一个 python 脚本，在最开始使用 yaml 库解析 lingling 的构建配置文件，然后依次生成 `git_source.tar.zst, control, postinst, postrm, deb.bash` 下面以玲珑包 bzip2 作为例子，依次介绍它们的作用：

- control

```ini
Package: linglong.bzip2
Version: 1.0.6
Architecture: amd64
Maintainer: wurongjie <wurongjie@deepin.org>
Depends: linglong.org.deepin.base (>= 23.0.0), linglong.org.deepin.base (<< 23.0.1)
Section: linglong
Priority: standard
Description: linglong to deb
```

  control 是 deb 包必须的文件，用于记录 deb 包的依赖、描述、包名等基础信息，其中包名、依赖和版本是从玲珑构建文件的信息转换而来的

- postinst

```sh
## Generated by linglong service
repoDir=${LINGLONG_REPO_DIR:=/var/cache/linglong/repo}
debDir=${LINGLONG_DEB_DIR:=/var/cache/linglong/deb}
mkdir -p $repoDir
ostree init --repo=$repoDir --mode=bare-user-only
ostree --repo=$repoDir commit --branch=linglong/bzip2/1.0.6/x86_64/devel --tree=tar=$debDir/linglong-bzip2-1.0.6-x86_64-devel.tar.zst
ostree --repo=$repoDir commit --branch=linglong/bzip2/1.0.6/x86_64/runtime --tree=tar=$debDir/linglong-bzip2-1.0.6-x86_64-runtime.tar.zst
```

  postinst 在 deb 安装时自动触发的脚本，用于安装玲珑包，这里将 deb 中附带的玲珑包导入到本地玲珑仓库，就算是安装成功了。压缩包的来源见 deb.bash 文件

- postrm

```sh
## Generated by linglong service
repoDir=${LINGLONG_REPO_DIR:=/var/cache/linglong/repo}
ostree --repo=$repoDir refs --delete linglong/bzip2/1.0.6/x86_64/devel
ostree --repo=$repoDir refs --delete linglong/bzip2/1.0.6/x86_64/runtime
```

  postrm 在 deb 卸载时自动触发的脚本，用于卸载玲珑包，这里将玲珑包从本地仓库删除就算是卸载了。压缩包因为是 deb 自动解压的，在 deb 卸载是会自动删除，不需要手动处理。

- deb.bash

```sh
#!/bin/bash -e
set -v on
# 创建deb打包目录
debDir=/var/cache/linglong/deb
mkdir -p DEB/DEBIAN DEB$debDir
# 复制control和构子脚本到deb打包目录
cp control postinst postrm DEB/DEBIAN/
chmod +x DEB/DEBIAN/postinst DEB/DEBIAN/postrm
# 将玲珑包导出成压缩包放到deb打包目录
ostree --repo=/home/abuild/.cache/linglong-builder/repo export bzip2/1.0.6/x86_64/devel | zstd -o DEB$debDir/linglong-bzip2-1.0.6-x86_64-devel.tar.zst
ostree --repo=/home/abuild/.cache/linglong-builder/repo export bzip2/1.0.6/x86_64/runtime | zstd -o DEB$debDir/linglong-bzip2-1.0.6-x86_64-runtime.tar.zst
# 使用dpkg-deb将deb打包目录打包成deb文件，并放到OTHER目录
dpkg-deb -b -Znone --root-owner-group ./DEB ../OTHER
```

用于打包出 deb 的脚本，在编译完玲珑包后，使用该脚本将玲珑封装成 deb 包，把 deb 包放到 OTHER 目录并修改 obs 项目的 Repotype 配置为 debian，obs 会自动创建 deb 仓库。

- git_source.tar.zst

因为玲珑自己支持从 git 仓库拉取源码，而 obs 构建是离线状态，需要提前把 git 仓库 拉取下来，放到 obs 源码目录，在构建时再解压使用。

其实这些文件的生成和 deb 打包也可以在构建过程中进行，但我觉得使用 `Source Service` 有以下几点好处：

- 更透明

Source Service 生成的文件可以在 web 界面直接查阅，构建过程中生成的文件只能从日志查看。

- 容调试

Source Service 是插件形式存在的，修改后不用重启 obs，在 web 界面能直接触发，obs 客户端还可以在本地运行 Source Service。

- 更自由

Source Service 可以使用任意语言编写，python 脚本或 go 这种静态编译都可以，只要是可执行文件就行，使用构建过程只能使用 bash 编写，没有 python 易用，并且 bash 不好解析 yaml 文件。

- 参数化

Source Service 是可以传递参数的，比如 linglong.yaml 文件的路径，这些都可以通过参数修改。

## 构建过程编写

构建过程是 bash 编写函数来实现，但是代码量并不多，可以查看[这个文件](https://github.com/myml/obs-build/blob/4738d13c4cdd8b9fdc30a428cf93f843a7f6992b/build-recipe-linglong)，这里只介绍两个主要函数：recipe_setup_linglong 和 recipe_build_linglong。

- recipe_setup_linglong

```bash
recipe_setup_linglong() {
    echo "setup linglong" $BUILD_ROOT
    # Copy from build-recipe-appimage
    TOPDIR=/usr/src/packages
    "$DO_INIT_TOPDIR" && rm -rf "$BUILD_ROOT$TOPDIR"
    for i in OTHER SOURCES DEBS ; do
	    mkdir -p "$BUILD_ROOT$TOPDIR/$i"
    done
    if test "$MYSRCDIR" = $BUILD_ROOT/.build-srcdir ; then
	    mv "$MYSRCDIR"/* $BUILD_ROOT$TOPDIR/SOURCES/
    else
	    copy_sources "$MYSRCDIR" "$BUILD_ROOT$TOPDIR/SOURCES/"
    fi
    chown -hR "$ABUILD_UID:$ABUILD_GID" "$BUILD_ROOT$TOPDIR"
    # init linglong repo
    mkdir -p /home/$BUILD_USER/.cache/linglong-builder
    mv /var/cache/linglong/repo /home/$BUILD_USER/.cache/linglong-builder/
    chown -hR "$ABUILD_UID:$ABUILD_GID" /home/$BUILD_USER/.cache/linglong-builder
}
```

从名字也能看出来，这个函数主要是进行预处理的，首先创建了一些构建时用到的目录，因为使用 deb 创建的本地玲珑仓库是 root 权限，而玲珑需要使用普通用户身份构建，就在 setup 函数中移动本地玲珑仓库到用户家目录，并改变仓库的归属。

- recipe_build_linglong

由于前期已进行了各种预处理，build 函数就很简单的，分别是解压源码、调用 ll-builder 离线构建和 deb 打包。

```bash
recipe_build_linglong() {
    echo "build linglong"
    su -c 'tar --no-same-owner -xf "git_source.tar.zst"' $BUILD_USER
    su -c '/usr/bin/ll-builder build --offline' $BUILD_USER || cleanup_and_exit 1
    su -c 'bash deb.bash' $BUILD_USER || cleanup_and_exit 1
    BUILD_SUCCEEDED=true
}

```

## 项目配置

通过以上改动，obs 就已经支持玲珑构建了，但是还需要在项目中添加配置，以要求 obs 在项目中使用玲珑构建。

```ini
# 构建类型
Type: linglong
# 依赖仓库类型
BinaryType: deb
# 发布仓库类型
Repotype: debian
# 预安装玲珑构建环境
Preinstall: linglong-builder
# 玲珑需要用 kvm 构建
Constraint: sandbox kvm
```

## 结束语

本文通过在 obs 集成玲珑构建来介绍怎么在 obs 添加新的包格式支持，希望能对你有所帮助。
