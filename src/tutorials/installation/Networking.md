# NixOS 的网络问题

国内用户在使用 NixOS 时会存在一些网络问题，一是 NixOS 高度依赖 GitHub 作为
channel/flake 数据源——在国内访问 GitHub 相当的慢，二是 NixOS 官方的包缓存服务器
在国内访问速度较慢。

为了解决这些问题，你可以使用国内的镜像源，或者使用代理工具来加速访问。

这里我先介绍几个比较简单的配置方法。

## 1. 使用国内的 Nix 包缓存服务器

首先，在执行后面给出的任何 `nix` 相关命令时，你都可以通过 `--option` 选项来指定
镜像源，例如：

```bash
# 使用上海交通大学的镜像源
# 官方文档: https://mirror.sjtu.edu.cn/docs/nix-channels/store
nixos-rebuild switch --option substituters "https://mirror.sjtu.edu.cn/nix-channels/store"

# 使用中国科学技术大学的镜像源
# 官方文档: https://mirrors.ustc.edu.cn/help/nix-channels.html
nixos-rebuild switch --option substituters "https://mirrors.ustc.edu.cn/nix-channels/store"

# 使用清华大学的镜像源
# 官方文档: https://mirrors.tuna.tsinghua.edu.cn/help/nix-channels/
nixos-rebuild switch --option substituters "https://mirrors.tuna.tsinghua.edu.cn/nix-channels/store"

# 其他 nix 命令同样可以使用 --option 选项，例如 nix shell
nix shell nixpkgs#cowsay --option substituters "https://mirrors.tuna.tsinghua.edu.cn/nix-channels/store"
```

你可以自己测试下上述几个镜像源的速度，选速度最快的一个。

## 2. 使用国内镜像地址加速 Flakes Inputs 的下载

如果你想使用 Flakes，但访问 GitHub 速度太慢，你可以使用国内的镜像地址来加速。

但需要注意的是，这种方式下无法锁定 nixpkgs 版本，也就失去了 Flakes 锁定依赖版本
的优势。

示例如下，主要是将 `nixpkgs.url` 替换成国内镜像源的 `nixexprs.tar.xz` 文件的路
径：

```nix
{
  inputs = {
    # nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11";
    nixpkgs.url = "https://mirrors.ustc.edu.cn/nix-channels/nixos-23.11/nixexprs.tar.xz";
    # nixpkgs.url = "https://mirrors.tuna.tsinghua.edu.cn/nix-channels/nixpkgs-23.11/nixexprs.tar.xz";
  };
  outputs = inputs@{ self, nixpkgs, ... }: {
    nixosConfigurations.my-nixos = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./configuration.nix
      ];
    };
  };
}
```

## 3. 使用代理工具加速访问 Channels 跟 Flake Inputs

对于 Flake Inputs 跟 Channels 的加速访问，这个就需要使用代理工具加速访问。

优先推荐使用旁路网关（软路由）或者 TUN 方式的全局网络加速方案，这是最省心的方
式。

如果你只有 HTTP 代理，可以通过如下命令设置代理环境变量，实现使用 socks5/http 代
理加速 nix 的网络访问：

```bash
sudo mkdir /run/systemd/system/nix-daemon.service.d/
cat << EOF >/run/systemd/system/nix-daemon.service.d/override.conf
[Service]
Environment="https_proxy=socks5h://localhost:7891"
EOF
sudo systemctl daemon-reload
sudo systemctl restart nix-daemon
```

**但请注意，系统重启后 `/run/` 目录下的内容会被清空，所以每次重启后都需要重新执
行上述命令**！

如果你希望永久设置代理，建议将上述命令保存为 shell 脚本，在每次启动系统时运行一
下。或者也可以使用旁路网关或 TUN 等全局代理方案。

更详细的说明与其他用法介绍，请移步
[添加自定义缓存服务器](https://nixos-and-flakes.thiscute.world/zh/nixos-with-flakes/add-custom-cache-servers)
，注意这部分内容可能需要一定的 NixOS 使用经验才能理解。

<!-- prettier-ignore -->
::: warning GitHub 报 HTTP 403 错误
注意：使用一些商用代理或公共代理时你可能会遇到 GitHub 下载时报 HTTP 403 错误
（[nixos-and-flakes-book/issues/74](https://github.com/ryan4yin/nixos-and-flakes-book/issues/74)），
可尝试通过更换代理服务器或者设置
[access-tokens](https://github.com/NixOS/nix/issues/6536) 来解决。

<!-- prettier-ignore -->
:::

## 4. 使用国内 Git 镜像替代 GitHub

由于 NixOS Flakes 依赖 GitHub 仓库进行代码托管，而国内访问 GitHub 可能较慢，建议使用国内的 Git 镜像（如 Gitee、清华大学开源镜像站等）来加速 Flake 依赖的获取。

### 4.1 替换 GitHub 为 Gitee 或其他国内镜像

在 Flake 配置文件 `flake.nix` 中，默认的 `nixpkgs` 可能使用 GitHub 作为数据源，例如：

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11";
  };
}
```

可以将其替换为国内的 Git 镜像，例如 Gitee：

```nix
{
  inputs = {
    nixpkgs.url = "git+https://gitee.com/mirrors/NixOS-nixpkgs.git?ref=nixos-23.11";
  };
}
```

### 4.2 直接修改 `nix.registry` 以自动使用国内 Git 镜像

可以通过 `nix registry` 机制，让所有 `github:NixOS/nixpkgs` 形式的引用自动跳转到国内镜像：

```bash
nix registry add nixpkgs git+https://gitee.com/mirrors/NixOS-nixpkgs.git
```

这样，所有 `github:NixOS/nixpkgs` 形式的 Flake 依赖都会被自动重定向到 Gitee，避免手动修改 `flake.nix`。

### 4.3 其他国内 Git 镜像来源

除了 Gitee 之外，还可以使用其他国内提供的 Git 镜像，例如：

- **清华大学开源镜像站**  
  - 地址：https://mirrors.tuna.tsinghua.edu.cn/help/git/  
  - 示例：
    ```nix
    nixpkgs.url = "git+https://mirrors.tuna.tsinghua.edu.cn/git/nixpkgs.git?ref=nixos-23.11";
    ```

- **中国科学技术大学开源镜像站**  
  - 地址：https://mirrors.ustc.edu.cn/  
  - 示例：
    ```nix
    nixpkgs.url = "git+https://mirrors.ustc.edu.cn/git/nixpkgs.git?ref=nixos-23.11";
    ```

### 4.4 手动克隆并使用本地 Git 仓库

如果仍然无法流畅访问 GitHub 或国内镜像，可考虑手动克隆 `nixpkgs` 到本地：

```bash
git clone --depth 1 -b nixos-23.11 https://gitee.com/mirrors/NixOS-nixpkgs.git ~/nixpkgs
```

然后在 `flake.nix` 中使用本地路径：

```nix
{
  inputs = {
    nixpkgs.url = "path:/home/your-username/nixpkgs";
  };
}
```

这种方法适用于 Git 访问受限的情况下，手动同步 `nixpkgs` 仓库，确保 Flake 解析不会因网络问题受阻。

---

通过上述方法，可以有效缓解国内用户在使用 NixOS 时因 GitHub 访问受限导致的 Flakes 下载问题，提高系统更新和软件包管理的效率。
