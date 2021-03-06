如何配置fail2ban来保护Apache服务器
================================================================================
生产环境中的Apache服务器可能会受到不同的攻击。攻击者或许试图通过暴力攻击或者执行恶意脚本来获取未经授权或者禁止访问的目录。一些恶意爬虫或许会扫描你网站下的任意安全漏洞，或者手机email地址或者web表格来发送垃圾邮件。

Apache服务器具有综合的日志功能来捕捉不同表明是攻击的异常事件。然而，它还不能系统地解析具体的apache日志并迅速地反应到潜在的攻击（比如，禁止/解禁IP地址）。这时候`fail2ban`可以解救这一切，解放了系统管理员的工作。

`fail2ban`是一款入侵防御工具，可以基于系统日志检测不同的工具并且可以自动采取保护措施比如：通过`iptables`禁止ip、阻止/etc/hosts.deny中的连接、或者通过邮件通知事件。fail2ban具有一系列预定义的“监狱”，它使用特定程序日志过滤器来检测通常的攻击。你也可以编写自定义的规则来检测来自任意程序的攻击。

在本教程中，我会演示如何配置fail2ban来保护你的apache服务器。我假设你已经安装了apache和fail2ban。对于安装，请参考[另外一篇教程][1]。

### 什么是 Fail2ban 监狱 ###

让我们更深入地了解fail2ban监狱。监狱定义了具体的应用策略，它会为指定的程序触发一个保护措施。fail2ban在/etc/fail2ban/jail.conf 下为一些流行程序如Apache、Dovecot、Lighttpd、MySQL、Postfix、[SSH][2]等预定义了一些监狱。每个依赖于特定的程序日志过滤器（在/etc/fail2ban/fileter.d 下面）来检测通常的攻击。让我看一个例子监狱：SSH监狱。

    [ssh]
    enabled   = true
    port      = ssh
    filter    = sshd
    logpath   = /var/log/auth.log
    maxretry  = 6
    banaction = iptables-multiport

SSH监狱的配置定义了这些参数：

- **[ssh]**： 方括号内是监狱的名字。
- **enabled**：是否启用监狱 
- **port**： 端口的数字 (或者数字对应的名称).
- **filter**： 检测攻击的检测规则
- **logpath**： 检测的日志文件
- **maxretry**： 禁止前失败的最大数字
- **banaction**： 禁止操作

定义配置文件中的任意参数都会覆盖相应的默认配置`fail2ban-wide` 中的参数。相反，任意缺少的参数都会使用定义在[DEFAULT]字段的值。

预定义日志过滤器都必须在/etc/fail2ban/filter.d，可以采取的操作在/etc/fail2ban/action.d。

![](https://farm8.staticflickr.com/7538/16076581722_cbca3c1307_b.jpg)

如果你想要覆盖`fail2ban`的默认操作或者定义任何自定义监狱，你可以创建*/etc/fail2ban/jail.local**文件。本篇教程中，我会使用/etc/fail2ban/jail.local。

### 启用预定义的apache监狱 ###

`fail2ban`的默认安装为Apache服务提供了一些预定义监狱以及过滤器。我要启用这些内建的Apache监狱。由于Debian和红买配置的稍微不同，我会分别它们的配置文件。

#### 在Debian 或者 Ubuntu启用Apache监狱 ####

要在基于Debian的系统上启用预定义的apache监狱，如下创建/etc/fail2ban/jail.local。

    $ sudo vi /etc/fail2ban/jail.local 

----------

    # detect password authentication failures
    [apache]
    enabled  = true
    port     = http,https
    filter   = apache-auth
    logpath  = /var/log/apache*/*error.log
    maxretry = 6
     
    # detect potential search for exploits and php vulnerabilities
    [apache-noscript]
    enabled  = true
    port     = http,https
    filter   = apache-noscript
    logpath  = /var/log/apache*/*error.log
    maxretry = 6
     
    # detect Apache overflow attempts
    [apache-overflows]
    enabled  = true
    port     = http,https
    filter   = apache-overflows
    logpath  = /var/log/apache*/*error.log
    maxretry = 2
     
    # detect failures to find a home directory on a server
    [apache-nohome]
    enabled  = true
    port     = http,https
    filter   = apache-nohome
    logpath  = /var/log/apache*/*error.log
    maxretry = 2

由于上面的监狱没有指定措施，这些监狱都将会触发默认的措施。要查看默认的措施，在/etc/fail2ban/jail.conf中的[DEFAULT]下找到“banaction”。

    banaction = iptables-multiport

本例中，默认的操作是iptables-multiport（定义在/etc/fail2ban/action.d/iptables-multiport.conf）。这个措施使用iptable的多端口模块禁止一个IP地址。

在启用监狱后，你必须重启fail2ban来加载监狱。

    $ sudo service fail2ban restart 

#### 在CentOS/RHEL 或者 Fedora中启用Apache监狱 ####

要在基于红帽的系统中启用预定义的监狱，如下创建/etc/fail2ban/jail.local。

    $ sudo vi /etc/fail2ban/jail.local 

----------

    # detect password authentication failures
    [apache]
    enabled  = true
    port     = http,https
    filter   = apache-auth
    logpath  = /var/log/httpd/*error_log
    maxretry = 6
     
    # detect spammer robots crawling email addresses
    [apache-badbots]
    enabled  = true
    port     = http,https
    filter   = apache-badbots
    logpath  = /var/log/httpd/*access_log
    bantime  = 172800
    maxretry = 1
     
    # detect potential search for exploits and php <a href="http://xmodulo.com/recommend/penetrationbook" style="" target="_blank" rel="nofollow" >vulnerabilities</a>
    [apache-noscript]
    enabled  = true
    port     = http,https
    filter   = apache-noscript
    logpath  = /var/log/httpd/*error_log
    maxretry = 6
     
    # detect Apache overflow attempts
    [apache-overflows]
    enabled  = true
    port     = http,https
    filter   = apache-overflows
    logpath  = /var/log/httpd/*error_log
    maxretry = 2
     
    # detect failures to find a home directory on a server
    [apache-nohome]
    enabled  = true
    port     = http,https
    filter   = apache-nohome
    logpath  = /var/log/httpd/*error_log
    maxretry = 2
     
    # detect failures to execute non-existing scripts that
    # are associated with several popular web services
    # e.g. webmail, phpMyAdmin, WordPress
    port     = http,https
    filter   = apache-botsearch
    logpath  = /var/log/httpd/*error_log
    maxretry = 2

注意这些监狱文件默认的操作是iptables-multiport（定义在/etc/fail2ban/jail.conf中[DEFAULT]字段下的“banaction”中）。这个措施使用iptable的多端口模块禁止一个IP地址。

启用监狱后，你必须重启fail2ban来加载监狱。

在 Fedora 或者 CentOS/RHEL 7中：

    $ sudo systemctl restart fail2ban 

在 CentOS/RHEL 6中：

    $ sudo service fail2ban restart 

### 检查和管理fail2ban禁止状态 ###

监狱一旦激活后，你可以用fail2ban的客户端命令行工具来监测当前的禁止状态。

查看激活的监狱列表：

    $ sudo fail2ban-client status 

查看特定监狱的状态（包含禁止的IP列表）：

    $ sudo fail2ban-client status [监狱名] 

![](https://farm8.staticflickr.com/7572/15891521967_5c6cbc5f8f_c.jpg)

你也可以手动禁止或者解禁IP地址

要用制定监狱禁止IP：

    $ sudo fail2ban-client set [name-of-jail] banip [ip-address] 

要解禁指定监狱屏蔽的IP：

    $ sudo fail2ban-client set [name-of-jail] unbanip [ip-address] 

### 总结 ###

本篇教程解释了fail2ban监狱如何工作以及如何使用内置的监狱来保护Apache服务器。依赖于你的环境以及要保护的web服务器类型，你或许要适配已存在的监狱或者编写自定义监狱和日志过滤器。查看outfail2ban的[官方Github页面][3]来获取最新的监狱和过滤器示例。

你有在生产环境中使用fail2ban么？分享一下你的经验吧。

--------------------------------------------------------------------------------

via: http://xmodulo.com/configure-fail2ban-apache-http-server.html

作者：[Dan Nanni][a]
译者：[geekpi](https://github.com/geekpi)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创翻译，[Linux中国](http://linux.cn/) 荣誉推出

[a]:http://xmodulo.com/author/nanni
[1]:http://xmodulo.com/how-to-protect-ssh-server-from-brute-force-attacks-using-fail2ban.html
[2]:http://xmodulo.com/how-to-protect-ssh-server-from-brute-force-attacks-using-fail2ban.html
[3]:https://github.com/fail2ban/fail2ban