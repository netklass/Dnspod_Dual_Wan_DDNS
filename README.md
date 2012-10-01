# DNSPod双线DDNS客户端 (DNSPod Dual Wan DDNS Client)
基于PHP, cUrl, DNSPod API的双线DDNS客户端  

## 章节0
不知道为什么突然想写这个章节0，总之想简单讲一下我写这个双线DDNS客户端的来源。

我在帮别人维护一个双线服务器，出于成本和业务量以及在线率需求的考虑，并没有进行服务器托管，而是同时拉了电信和联通两根ADSL进来接到服务器里。

最初的架构是服务器双网卡，同时接两个TP-Link路由器，每个路由器里都设置好花生壳的DDNS。  

可是不知道是TP-Link不给力还是花生壳不给力，总之就是三天两头的掉线，而且掉线后还不会自动重连。后来还试过在服务器里又安了一个meibu的DDNS客户端当备用，但是由于线路选择的问题，效果总是不理想。

于是就购买了花生壳的VIP，结果没有理解透花生壳的服务方式：如果你是双线的话，购买了VIP只能给一根线路使用，要给另一根线路使用还得再购买一个VIP。更可惜的是在购买了花生壳的VIP后，花生壳的VIP服务就无法连接。于是以服务不可用，并且VIP服务并不是我想要的为理由要求退款。  

但是，你永远不能低估国内厂家的下限，在最初的几次沟通里，花生壳完全无视他7天退款的条约，也无视我购买后无法使用的问题，只是抓住我理解错了VIP的用途，认为我要求退款的理由不充分，拒绝退款。甚至我主动要求承担手续费都不退款。最终，经过了30多封邮件，甚至连律师函都发过去后，他们终于扭扭捏捏的退款了，而且还扣了手续费。我早都给他们在邮件里说清楚了我肯定是要退款的，没有缓和的余地的，结果他们扭扭捏捏了这么半天，最终还是退款了（还扣了手续费）。唉，贱就是一个字。于是完全抛弃了花生壳，开始寻找下一个解决方案。

接下来发现了DNSPod自己的DDNS客户端也可以绑定网卡，于是下载试用之。希望越大，失望越大啊：DNSPod自己的DDNS客户端绑定网卡老不成功（即绑定的网卡A，但是有时候还是从网卡B走出去的）不说，连接失败到一定次数后就直接放弃重试了。于是继续寻找下一个解决方案。

中途还试过开两个虚拟机分别桥接两块网卡进行DDNS，但是开销有些大，就放弃了。

再后来发现wget可以绑定网卡，于是这个双线DDNS客户端最初的版本就是用在Windows里安装了个Wget，然后在PHP里调用系统命令，运行Wget获取IP，别说这个方法还的确有效，代码良好运行了一段时间。

再后来突然发现php里的curl扩展也可以绑定网卡，于是就有了curl版的DDNS客户端（毕竟可以少安装一个wget），但是最初的时候为了省两个curl连接（外加偷懒），每次获取完IP后不管IP是否有变化，都更新到DNSPod里。刚开始工作的也挺不错的，直到我以1分钟三条日志的速度，撑爆了DNSPod后台允许查看的最大日志数后，DNSPod增加了这块的限制，在一小时内调用API更新次数达到一定限制后就不允许继续调用了。于是就继续改写，就有了当前这个版本的DDNS客户端了。

## 设置说明
用记事本打开源代码后，只需要修改前面几行
> `USERNAME` 就是在DNSPod里的用户名  
> `PASSWORD` 就是在DNSPod里的密码  
> `DOMAIN_ID` 就是域名的ID，在后面我会说明获取的方法  
> `SUB_DOMIAN` 就是你要设置DDNS的二级域名，如果要针对一级域名设置DDNS，则填写为 @  
> `IP_API` 就是获取IP地址的接口，默认的是我的博客服务器，服务器在香港，绝大多数情况下都能工作的很良好，但是也不能排除线路抽风的情况。所以有条件的也可以改成自己的服务器地址。返回格式就直接是IP地址就行了，不需要用xml/json进行包装  
> `dx_interface` 就是连接电信线路网卡的内网IP，一定要为静态IP。因为curl是根据IP来绑定网卡的，而不是MAC地址或者网卡名称  
> `lt_interface` 就是连接联通线路网卡的内网IP，同样也需要为静态IP，理由同上  
> `record_id` 三条DNSPod记录的id，在后面我会说明获取的方法

设置完成后保存，用 **Windows计划任务** 或者 **crontab** 定时每分钟运行一次就行了。

## 其他说明
* 理论上讲这段代码在 Windows / Linux / Mac 里都可以运行，不过暂时没有条件在后两者里试验（需要双网卡连接外网啊有木有）  
* 这段代码在实际运行的时候会遇到以下问题：比如说客户端正在通过电信线路连接服务器，突然电信线路掉线，代码检测掉电信线路无法连接到外网后就会把电信的连接也指向到网通IP，这是客户端立刻重试，就会连接到网通的IP，接着电信一会又重新拨号成功了，代码又把电信的连接指向电信的IP，但是客户端不会主动断开网通IP的连接而连回电信的IP，就会造成客户端感觉断线了，再连就变卡了，但是看服务器状态却又是正常的。这点暂时无解，只能说他们发现断线后稍等5分钟再连，或者发现连接太慢后重新连接一下。
* 由于代码运行的时间差，以及默认ttl的原因，IP更改后最长可能需要3分钟才会被客户端获取到新IP。

## DOMAIN\_ID获取说明
使用Internet Explorer 8 / 9登录DNSPod网站，进入到域名列表，这时按下 **F12**，再按下 **Ctrl + B**，选中你要设置DDNS域名前面的选择框，这时下方的代码框会高亮出类似这样的一句代码

`<input name="check" type="checkbox" value="1357924"/>`

其中的 **`1357924`** 就是你的DOMAIN\_ID了。

## record\_id获取说明
接着上步，进入到域名的解析记录列表，请先随意添加三条记录（如果已经添加过就不用再添加了），添加完成后点下页面下方 **开发人员工具** 里的刷新按钮，再按下 **Ctrl + B**，选中你刚才添加的记录前的选择框，这是下面的代码框会高亮出类似这样的一句代码

`<input name="check" class="record-checkbox" type="checkbox" value="12345678"/>`

其中的 **`123456789`** 就是record\_id了。  
接着再重复上面的操作，按下 **Ctrl + B** ，选择另一条记录前的选择框，获取record\_id，接着再重复，把三个record\_id获取出来。

## 最后说明
* 以上的两步操作也可以用Firefox + Firebug / Chrome / Opera完成，用IE做解释是因为大家都有IE（无视水果机）
* 这段代码是根据现在正在运行的代码添加注释，美化代码重写出来的（原版代码无注释，代码写的很乱）。原版代码已经稳定运行了一年多，这个重写的应该也没有太大问题。
* 如果以后哪天闲的无聊了，或许会写一个DOMAIN\_ID / record_id 获取工具出来。但是，等着吧...
* 如果在使用时遇到了任何问题，欢迎和我交流：
> QQ: 565837499  
  Email: <vibbow@gmail.com>  
  Blog: <http://vsean.net/>