JetBrains 系列 IDE 一直是我开发的主力工具，在开发时往往选择在本机进行运行和调试。这样毫无疑问是很高效的开发方式，但有时我们希望在更接近于线上的环境中进行调试，那么如何使此次的修改快速见效(部署以及运行)是需要解决的关键问题之一。

JetBrains GoLand、JetBrains WebStorm 和 IntelliJ IDEA 是我用得最多的 IDE，接下来以一个用 go 开发的前后端分离的网站为例说明如何快速使修改在服务器上部署并见效。

#### 准备并配置好服务器
项目的前端部分将部署在服务器 A 上，使用 nginx 进行代理，后端部分部署在 B 服务器上，前端通过 RESTful api 的方式调用后端。

服务器 A 需要装好 nginx，配置监听 80 端口，这里以[我的个人网站](http://www.duanjn.com)为例:
```shell
root@vultr:~# cat /etc/nginx/sites-available/my 
server {
	listen 80;

	server_name www.duanjn.com duanjn.com;
        root /usr/duan/nginx/sites/www/duanjn.com;
        index index.html;

	location / {
                 try_files $uri $uri/ =404;      
	}

        access_log /usr/duan/nginx/logs/my.access.log;
        error_log /usr/duan/nginx/logs/my.error.log;
}
```
可见网站的静态资源保存在 `/usr/duan/nginx/sites/www/duanjn.com` 目录下。此外还需将域名解析到服务器 A。

服务器 B 用于运行后端项目，需要提前安装好 go。稍后我们可以选择上传源码或上传可执行程序到服务器的方式进行部署，可执行程序可以选择任意位置或 `/root/go/bin`目录(上传可执行程序时需要先在本地 build，build 时注意构建环境 GOARCH，GOOS 要指定为服务器的)。这里我选择上传源码的方式，上传目录在这: `/root/go/src/mysite`
#### 配置 IDE 连接远程服务器
JetBrains 系列 IDE 都提供了 Tools -> Deployment 工具，用于快捷的将本地文件上传到服务器。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191212101207375.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FpbWVpbWVpVFM=,size_16,color_FFFFFF,t_70)
先说后端(JetBrains GoLand)的配置，点击 Configuration 进行配置:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191212101353596.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FpbWVpbWVpVFM=,size_16,color_FFFFFF,t_70)
注意 Root path 指的是挂载到的服务器上的目录。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191212101628508.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FpbWVpbWVpVFM=,size_16,color_FFFFFF,t_70)
Local path 指定本地文件的目录，上传时该目录下的所有文件都将被上传到服务器。Deployment path 为上传文件后服务器上保存这些文件的位置，注意这个目录是基于 Root path 的相对路径，即如果 Root path 为 `/usr/duan`，那么最终的目录就是 `/usr/duan/root/go/src/mysite`，所以上一步我把 Root path 设置为 `/`

JetBrains GoLand 和 JetBrains WebStorm 的配置方式都是类似的，不同的是服务器部分的配置：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191212102508246.png)
需要配置对应的服务器。

以及路径映射部分，JetBrains WebStorm 配置如下:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191212102651642.png)

#### 部署(上传)并运行
上一步配置好后 IDE 就会显示上传按钮:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191212102835624.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FpbWVpbWVpVFM=,size_16,color_FFFFFF,t_70)
点击 Upload to **，文件就会上传到服务器。对于前端，因为我们已经配置并启动了 nginx，只要文件上传成功，刷新页面就能看到最新的效果。

而后端部分我们选择上传源码的方式，因此上传成功之后需要在 `/root/go/src/mysite` 下依次执行如下命令运行项目:
```shell
go get -v
go build
./mysite # 直接前台启动，而不是后台启动，这样下次新的改动上传需要重新启动时只需 ctrl+C 后重新执行这三个命令即可
```
此外对远程运行的 go 项目，如果想在本地进行 debug，可以借助 [dlv+GoRemote](https://juejin.im/entry/5d5ce39ef265da039a288b85) 完成。

Tips
- ssh 客户端我使用的是 XShell，IDE 也提供了工具: Tools -> Start SSH session
- 对于 java 项目我们可以以类似的方式进行部署调试，不同的是可以选择上传 jar 或 war 的方式进行部署。
- 除了上述的方式外还可以借助 Git 将最新代码从本地 push 到仓管，然后服务器再从仓库 pull 最新的
- 当然还有一种方式是手动上传文件，推荐借助 [lrzsz](https://blog.csdn.net/wuzhangweiss/article/details/79842283) 工具
