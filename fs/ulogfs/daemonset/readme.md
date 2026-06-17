# daemonset
每台物理机只有一个ulogfs daemonset、一个logtail daemonset。
ulogfs daemonset及logtail daemonset共用一块大的tmpfs

## ulogfs daemonset
包含一个ulogfsManger及若干ulogfs进程。
- 创建一个mount point就生成一个ulogfs进程来响应
- ulogfsManager负责响应apiServer的请求，管理ulogfs进程，包括mount point的新建与销毁
  - mount point创建
  - mount point销毁

## logtail daemonset
包含一个logtailManager进程及一个logtail进程
