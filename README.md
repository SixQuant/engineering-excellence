# Engineering Excellence
工程卓越（精益求精）：软件开发运行过程中用到的各种环境以及工具的正确使用说明！

# Docker
- [Docker for Mac/Windows](docker/docker-install-mac-vm-centos.md)<br>
  开发环境中有时候想把 Docker 容器实例当做正常的虚拟机来用，换句话说就是本机和Docker容器实例处于同一个子网中，本机可以直接通过IP地址访问Docker容器实例，而不是通过中间端口映射的方式来访问！

* [Docker Issue Network Delay](docker/docker-issue-network-delay.md)<br>
  Docker的自定义网络启动会延迟大概40秒！如果你使用自定义网络在一个容器启动时想访问另外一个容器会失败！但是如果你先等待40秒再访问的话就一切正常！
