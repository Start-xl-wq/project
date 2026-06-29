# 作为device
## device->host
echo "hello windows" > /dev/ttyGS0 
会直接显示到pc 串口工具
## host->deivce
* ls -l /dev/ttyGS* 确保设备节点存在
* stty -F /dev/ttyGS0 raw -echo 配置为tty
* cat /dev/ttyGS0  监听输入
