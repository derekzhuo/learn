# sidecar方案
一个pod里面有三个container，一个业务container，一个ulogfs container，一个logtail container；卷三个container都可见
约束：
- biz container需要待ulogfs及logtail container启动之后才能启动，特别是要等ulogfs的挂载点准备好
- ulogfs根据传入的参数确定mount point及后端真实目录
  - 这里需要处理一下后端真实目录，可以通过环境变量或者cmdline参数传入
- logtail从环境变量里获取到mount point及真实的采集目录

## biz container
- 只感知业务日志目录及文件

## ulogfs container
- 生成mount point
- 拦截日志io、写到tmpfs

## logtail container
- 采集数据
- 发送sparse给ulogfs