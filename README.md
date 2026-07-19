# linana

> 极简、零依赖的容器运行时，纯 Bash 实现（220LOC）。

[![License](https://img.shields.io/badge/license-MIT-pink)](./LICENSE)

---

## 🐱 这是什么

linana 是一个极简的容器运行时，纯 Bash 实现。

~~也可以叫李娜娜（划掉）~~

> 这是我的 Bash 毕业作，一个仅 220 LOC 的容器运行时。

> 它的简洁并非来自删减功能，而是来自选择正确的抽象。

**与 ruri/Droidspaces/Docker 的区别：**

linana 不是这些神作的替代品。它是你在形如 Android 一般的受限环境时想跑容器，不想折腾受限内核，也不想为了简单需求而折腾更深入时的好帮手。

- 只对已有的 rootfs 镜像进行操作。
- 只用 **mount + UTS** 两个命名空间，较高兼容性。
- 适配 Android，自动提权和加载 Termux 环境，  
  允许通过命令参数挂载 /storage 存储。
- 使用 rootfs 文件路径的 hash 做引用寻址，实现无状态管理。

## 🧬 为什么只有 220 LOC？

linana 并不是一个容器平台，而是一个 Runtime。

它不维护 daemon；
不保存数据库状态；
不重新实现 Linux 已有的机制。

Namespace、Mount、Process 都直接交给 Linux 管理，
linana 只负责将它们组合成一个一致、可预测的 CLI。

220 LOC 并不是目标，而是这种设计自然得到的结果。

![usage](./usage.png)

---

linana 的目标不是实现一个功能堆叠的容器平台，而是在受限的 Android 环境中，用最少的抽象实现一个真正可用的容器运行时。

它刻意避免引入不必要的复杂度：

- 不维护后台 daemon；
- 不依赖数据库保存容器状态；
- 不模拟 Docker API 或额外抽象层；
- 不要求修改 Android 内核配置。

Runtime 之外的复杂度，都刻意留给 Linux 自己表达。

linana 不保存状态，而是让 Linux 本身成为状态来源。

- namespace 生命周期由内核管理；
- mount 状态由挂载树表达；
- 进程生命周期决定容器生命周期。

容器实例采用基于 rootfs 路径的内容寻址设计，通过 hash 自动生成唯一引用，实现无需数据库的无状态管理。

220 LOC 并不是追求极限压缩代码，而是边界裁剪后的自然结果。
当 Runtime 只负责 Runtime，本不属于它的复杂度就不需要存在。

它使用最少的代码，将 rootfs 引用、namespace 创建、mount 生命周期、容器进入以及 CLI 状态机组合成一个完整运行时。

代码结构遵循 Unix 工具设计理念：

- 清晰的命令分层；
- 可预测的状态转换；
- 最小化外部依赖；
- 通过现有 Linux 工具组合复杂能力。

最终得到的不是一个“大而全”的容器平台，而是一个小巧、透明、容易审计和维护的 Android 原生 chroot runtime。
它更像一个 Runtime，而不是一个 Platform。

---

**实现 start / enter / restart / exec ... 那些子命令与无状态管理的说明大概如下：**

```plaintext
    unshare -m -u sleep infinity    参数指定 IMGPATH（非必选）
           ↓                                 ↓
          PID                      sha256(IMGPATH) 取前八位 hash
           ↓                                 ↓
           │                   ┌─ PID 文件: /tmp/lina_<hash>.pid
           │                   └─ 挂载点:   /mnt/lina/<hash>
           │                                 ↓
           │                   mount $IMG → /mnt/lina/<hash>
           ↓                                 ↓
    nsenter 进入命名空间 ────────────────────┘
           ↓
    bind mount: /dev /dev/pts /proc /sys
    tmpfs:      /tmp /run
    (可选 Android: rbind /storage/emulated/0)
           ↓
    chroot /mnt/lina/<hash> → 容器就绪

    enter/exec:  nsenter → chroot → /bin/su - .../-c ...
    top:         遍历 /proc，按 mntns 过滤进程
    ps:          遍历 /tmp/lina_*.pid，检查存活 → 列出所有容器

    stop:
      kill 命名空间内所有进程 (按 mntns 匹配)
           ↓
      kill sleep (内核自动回收命名空间)
           ↓
      umount /mnt/lina/<hash>
           ↓
      rm pidfile
```

## 😺 快速开始

```plaintext
...
```

## 🐭 命令一览

~~先放个 usage 在这~~

```console
Usage: linana [Options] <Command> [args]

Options:
  -i, --img <PATH>          Specify the path to the rootfs image
  -S, --android-storage     Mount Android internal storage (/storage/emulated/0) into the container (Android only)

Commands (manage a specific container -- requires --img):
  start                     Start the container (idempotent: no-op if already running)
  stop                      Stop the running container and clean up
  restart                   Stop (if running) then start the container
  enter [USER=root]         Start the container if needed, then launch a login shell
  exec <command...>         Execute a command inside a running container
  top                       List topesses inside the running container

Commands (manage all containers -- no --img needed):
  ps                        List all containers (id, pid, mntns, img, mp)
  help                      Show this help message
```

## 🐰 Termux / Android

在 Termux 里会自动提权，保留 Termux 环境变量。  
依赖 su --mount-master，在标准 Linux 环境时提示使用 sudo。

可选参数 `--android-storage` 会把 `/storage/emulated/0` 以 rbind 方式暴露进容器。

## 🦉 其它

使用 `shfmt -i 4 -ci -sr -s -d` 格式化，保持较高可读性。

## ⚖️ LICENSE

[MIT](./LICENSE)
