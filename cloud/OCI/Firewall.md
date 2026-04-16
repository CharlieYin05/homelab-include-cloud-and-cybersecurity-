# OCI default iptable

ubuntu@cy-server-ora-arm:~$ sudo iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     0    --  0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
ACCEPT     1    --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     0    --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     6    --  0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
REJECT     0    --  0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
**REJECT     0    --  0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited** (坑爹所在）

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
InstanceServices  0    --  0.0.0.0/0            XXX.XXX.XXX.XXX/XXX      

(OCI内部服务专用通道↓)
Chain InstanceServices (1 references)
target     prot opt source               destination         
ACCEPT     6    --  0.0.0.0/0            169.XXX.XXX.XXX          owner UID match 0 tcp dpt:3260 /* See the Oracle-Provided Images section in the Oracle Cloud Infrastructure documentation for security impact of modifying or removing this rule */
.......
