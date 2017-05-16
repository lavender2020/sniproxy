防污染 DNS 和 SNI Proxy
======================

# 1. 组件简介
## Dnsmasq
具有 DHCP Server、DNS Server、DNS Forwarding、TFTP Server、PXEboot Server 等各种功能。主要使用其 DNS 转发功能
## 1.1. SNI
是 TLS 协议的扩展，在tcp握手开始的时候，告知服务器目标网站的Hostname，这样一个https服务器上面就可以部署多个具有不同证书的网站，根据目标网站的Hostname而采用不同的证书。
其实HTTP/1.1很早就有了这种 “Virtual Host”(虚拟主机)的概念，SNI 让HTTPS也支持
## 1.2. SNI Proxy
HTTPS代理软件，和基本的nginx反代原理相同。专门设计用来代理HTTPS站点，并可以独立运行，而不依赖任何第三方软件。
# 2. 防污染DNS
## 2.1. 基本逻辑
vps1 运行 dnsmasq，监听53和5353端口，TCP/UDP 都监听，上游 DNS 为 8.8.8.8、8.8.4.4  
vps2 运行 dnsmasq，上游为 vps1的地址，使用5353端口，并配置 dnsmasq-china-list，也就是把常见国内站点使用国内dns解析，列表外的站点查询上游DNS
有些人的方案是，仅使用一个国内vps，使用 DNSCrypt 连接 opedns 等提供加密通道的上游DNS，也是不错的
## 2.2. Dnsmasq 的基本配置

/*   
 * filename: /etc/dnsmasq.conf   
 */  

interface=eth0  
interface=lo  
    \# 上面是你想要监听的网络接口
 
conf-dir=/etc/dnsmasq.d/  
    \# 这里加载 /etc/dnsmasq.d/ 目录里面的一些列表, 待会我们会放一些 dnsmasq-china-list 列表  
 
    \#上游DNS   
no-resolv  
    \# 不使用系统 /etc/resolv.conf 里面的配置  
 
server=10.0.0.1#5353  
    \# 这里配置上游服务器，比如 10.0.0.1, 端口为5353  
    \# 用 5353 端口为了防止污染  


## 2.3. iptables 转发5353端口的查询
上面 dnsmasq 默认监听的是53端口的 TCP/UDP 连接，下面我们通过 iptables 将5353端口的查询转发至53端口，这个可以用来解决部分 ISP 的  dns 污染问题。（看个人需要，是否在两个vps上面都打开5353端口，毕竟一些操作系统的网络设置不支持 DNS 自定义端口…

iptables -t nat -A PREROUTING -p tcp --dport 5353 -j REDIRECT --to-port 53  
iptables -t nat -A PREROUTING -p  udp --dport 5353 -j REDIRECT --to-port 53

## 2.4. 国内站点加速
方案一：准备 被墙网站 的列表，统一查询国外 dns，其余的走国内DNS；参考 gfwlist
方案二：准备 国内站点 列表，这部分走国内，其他的查询国外 dns；参考 dnsmasq-china-list

git clone https://github.com/felixonmars/dnsmasq-china-list  
    \# clone 一下前人的成果  
    
    \# 下面建立三个软链接  
sudo ln -s `pwd`/dnsmasq-china-list/accelerated-domains.china.conf /etc/dnsmasq.d/  
sudo ln -s `pwd`/dnsmasq-china-list/bogus-nxdomain.china.conf /etc/dnsmasq.d/  
sudo ln -s `pwd`/dnsmasq-china-list/google.china.conf /etc/dnsmasq.d/  

git clone https://github.com/felixonmars/dnsmasq-china-list  
    \# clone 一下前人的成果  
 
    \# 下面建立三个软链接就好啦  
sudo ln -s `pwd`/dnsmasq-china-list/accelerated-domains.china.conf /etc/dnsmasq.d/  
sudo ln -s `pwd`/dnsmasq-china-list/bogus-nxdomain.china.conf /etc/dnsmasq.d/  
sudo ln -s `pwd`/dnsmasq-china-list/google.china.conf /etc/dnsmasq.d/  


下面这样的配置条目，指定特定泛域名的上游服务器  

server=/163.com/114.114.114.114(#端口号)  

你想让国内站点使用适合你的 DNS服务器进行查询，可以通过该项目提供的脚本实现一键替换  

cd /path/to/your/dnsmasq-china-list  
./dnsmasq-update-china-list ali  
    \# 这样就把那些记录里面的DNS地址都改为阿里的 223.5.5.5 了  
    \# 还有其他一些常用的地址，您可以查看这个脚本  
 
./dnsmasq-update-china-list 11.11.11.11  
    \# 这样就是使用指定的 dns 服务器地址  

## 2.5. SNI Proxy 站点的NS记录
方案一，泛域名  
按照下面的配置，可以将某个泛域名统一解析到相应的 IP 上  

/*  
 * filename: /etc/dnsmasq.d/sni_hosts.conf   
 */  

address=/google.com/192.168.1.1  
address=/gmail.com/192.168.1.1  
address=/gstatic.com/192.168.1.1  


方案二，单域名  
如果不需要进行泛解析，只要对几个单域名进行解析，也可以像下面这样  

echo "addn-hosts=/etc/my_hosts" >> /etc/dnsmasq.conf  


/etc/my_hosts 文件内容的书写格式同系统的 /etc/hosts  

# 3. SNI Proxy
## 3.1. 安装

yum install autoconf automake curl gettext-devel libev-devel pcre-devel perl pkgconfig rpm-build udns-devel -y  
yum install sniproxy -y  

## 3.2. 配置
vi /etc/sniproxy.conf  



\# sniproxy example configuration file  
\# lines that start with # are comments  
\# lines with only white space are ignored  
 
user daemon  
 
\# PID file  
pidfile /var/run/sniproxy.pid  
 
resolver {  
    nameserver 8.8.8.8  
}  
 
error_log {  
    \# Log to the daemon syslog facility  
    \#syslog daemon  
 
    \# Alternatively we could log to file  
    filename /var/log/sniproxy.log  
 
    \# Control the verbosity of the log  
    \#priority notice  
    priority debug  
}
 
access_log {  
    \# Same options as error_log  
    filename /var/log/sniproxy/sniproxy-access.log  
}  
 
listen 443 {    
    proto tls  
    table https_hosts  
}  
 
listen 465 {  
    proto tls  
    table xmpp_imap_smtp  
}  
 
listen 993 {  
    proto tls  
    table xmpp_imap_smtp  
}  
 
listen 995 {  
   proto tls  
   table xmpp_imap_smtp  
}  
 
listen 5222 {  
    proto tls  
    table xmpp_imap_smtp  
}  
 
listen 5223 {  
    proto tls  
    table xmpp_imap_smtp  
}  
 
listen 5269 {  
    proto tls  
    table xmpp_imap_smtp  
}  
 
\# named tables are defined with the table directive  
table https_hosts {  
 
    \# WordPress  
    (.*\.|)wp\.com$ *  
    (.*\.|)w\.org$ *  
    (.*\.|)wordpress\.com$ *   
    (.*\.|)gravatar\.com$ *  
 
    \# Wikipedia  
    (.*\.|)wikipedia\.org$ *  
 
    \# Twitter  
    (.*\.|)twimg\.com$ *  
    (.*\.|)tinypic\.com$ *  
    (.*\.|)twitpic\.com$ *  
    (.*\.|)twitter\.com$ *  
    (.*\.|)tweetdeck\.com$ *  
    (.*\.|)t\.co$ *  
 
    \# Facebook  
    (.*\.|)facebook\.com$ *
    (.*\.|)fbstatic\.com$ *
    (.*\.|)fbcdn\.net$ *
 
    \# Flickr
    (.*\.|)flickr\.com$ *
    (.*\.|)staticflickr\.com$ *
 
    \# bit.ly
    bitly\.com$ *
    bit\.ly$ *
 
    \# Google
    (.*\.|)googleapis\.com$ *
    (.*\.|)google\.com$ *
    (.*\.|)google\.co\.jp$ *
    (.*\.|)google\.com\.hk$ *
    (.*\.|)google\.com\.tw$ *
    (.*\.|)youtube\.com$ *
    (.*\.|)ytimg\.com$ *
    (.*\.|)googlevideo\.com$ *
    (.*\.|)googlehosted\.com$ *
    (.*\.|)googleusercontent\.com$ *
    (.*\.|)ggpht\.com$ *
    (.*\.|)gstatic\.com$ *
    (.*\.|)googlemail\.com$ *
    (.*\.|)googlecode\.com$ *
    (.*\.|)googledrive\.com$ *
    (.*\.|)blogspot\.com$ *
    (.*\.|)appspot\.com$ *
    (.*\.|)gmail\.com$ *
    (.*\.|)googlezip\.net$ *
    (.*\.|)googlesource\.com$ *
    (.*\.|)g\.cn$ *
    (.*\.|)google\.cn$ *
 
    \# ingress
    (.*\.|)panoramio.com$ *

    \# autodraw
    (.*\.|)autodraw\.com$ *
 
    \# Imgur
    (.*\.|)imgur\.com$ *
 
    \# Amazon AWS
    (.*\.|)amazonaws.com *
 
    \# CDN
    github\.global\.ssl\.fastly\.net *
    cdn\.sstatic\.net *
}
 
table xmpp_imap_smtp {  
    (.*\.|)google\.com$ *  
    (.*\.|)googlemail\.com$ *  
    (.*\.|)gmail\.com$ *  
}  



## 3.3. 启动守护进程
sniproxy  -c /etc/sniproxy.conf  

## 3.4. http 重定向到 https
因为我们的SNI Proxy 只针对 HTTPS 站点，所以如果用户发起HTTP请求，我们必须做好跳转。  
在 nginx 的 默认站点配置文件里进行如下修改：  


server {  
        listen   80 default_server;;  
        server_name _;     
            \# server_name 这样配置代表匹配任何virtual host里都未配置的域名  
        location / {  
                rewrite ^ https://$host$request_uri permanent;  
        }  
}  


