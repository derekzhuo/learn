# daemonset
每台物理机只有一个ulogfs daemonset、一个logtail daemonset。
ulogfs daemonset及logtail daemonset共用一块大的tmpfs

## ulogfs daemonset
包含一个ulogfsManger及若干ulogfs进程。
- 一个ulogfs响应一个mount point
- ulogfsManager负责响应apiServer的请求，管理ulogfs进程，包括mount point的新建与销毁
  - mount point创建
    - 创建业务pod的时候，apiServer发起一个tcp请求给ulogfsManager
      - ulogfsManager启动一个ulogfs，ulogfs创建mount point，同时准备好后端目录
      - ulogfsManager http请求logtailManager，表达需要创建一个mount point，需要携带的信息包括
        - mount point
        - 后端真实的目录
  - mount point销毁
    - apiServer向ulogfsManger tcp请求，准备销毁mount point
    - ulogfsManager请求logtailManager，确认当前数据是否全部采集完成，确定之后开始销毁
      - 关闭ulogfs进程

## logtail daemonset
包含一个logtailManager进程及一个logtail进程
- 只需要一个进程
  - 因为不需要像ulogfs那样感知多个mount point，只需要感知mount point与真实目录的映射即可
  - sparse差异
    - sidecar：logtail进程只感知一个ulogfs，这个可以写死的
    - daemonset：内部有多个ulogfs进程，其tcp端口无法确认，除非与mount point或者真实的目录绑定
      - 方案：感知ulogfsManager暴露的tcp端口，由ulogfsManager来处理
        - ulogfsManager收到sparse请求后根据fileName（绝对路径）来确认是哪个ulogfs
        - ulogfsManager向ulogfs转发sparse请求
