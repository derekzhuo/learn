# sidecar
一个pod里面有三个container，一个业务container，一个ulogfs container，一个logtail container；卷三个container都可见
约束：
- biz container需要待ulogfs及logtail container启动之后才能启动，特别是要等ulogfs的挂载点准备好
- ulogfs根据传入的参数确定mount point及后端真实目录

## biz container
- 只感知业务日志目录及文件

## ulogfs container
- 处理一个mount point
- 数据

## logtail container