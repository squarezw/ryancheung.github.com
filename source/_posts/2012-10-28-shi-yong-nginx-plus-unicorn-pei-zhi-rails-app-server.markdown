---
layout: post
title: "使用 Nginx+Unicorn 配置 Rails App Server"
date: 2012-10-28 13:16
comments: true
categories: rails unicorn nginx
---

Unicorn 是一个非常不错的 Ruby HTTP Server。它是被设计用来给客户端提供低延迟、
高带宽连接的服务，而且它利用了 Unix/类Unix 内核的特性优势。
目前很多使用 Rails 的大型网站如 Twitter, Github 等都在使用它。

下面我想先从几个方面讲下 Unicorn 的设计，然后在看怎么配合 Nginx 配置 Rails server。

## 设计

{% img /images/angry_unicorn.png %}

Unicorn 中文及独角兽或麒麟的意思，名字听起来就很霸气。下面是它的架构:

{% img /images/unicorn_arthitechture.png %}

Nginx 通过一个共享的 Unix Domain Socket(或者 TCP) 将请求重定向到 Unicorn 的 worker 进程池。
Unicorn 的 master 进程则管理这些 worker 进程并且由操作系统来处理负载均衡。
而 master 进程自己不负责任何请求。每个 master 可以使用不同版本的 Ruby 启动，
因此同台机器上其每个 Rails app 可以使用不同版本的 Ruby。

下面是个 nginx.conf 部分配置:

    upstream app_server {
      # for UNIX domain socket setups:
      server unix:/tmp/.sock fail_timeout=0;

      # for TCP setups, point these to your backend servers
      # server 192.168.0.7:8080 fail_timeout=0;
      # server 192.168.0.8:8080 fail_timeout=0;
      # server 192.168.0.9:8080 fail_timeout=0;
    }

当 Unicorn master 进程启动时，它把 app 和 gems 加载到内存。
完成后再 fork 出 worker 进程。不同 worker 进程间的负载均衡由操作系统负责。
所有 workers 共享一套监听 sockets 并且对他们调用非阻塞 accept()，
系统内核决定把 socket 给哪个 worker。如果没有 socket 需要 accept()，worker 将 sleep。

### 慢请求

Unicorn master 进程准确知道每个 worker 处理一个请求需要多久。
如果某个 worker 超过30秒(默认配置为30秒)响应请求，
master 进程会立即 kill 掉这个 worker 并且 fork 出一个新的，
这个新创建的worker 可以立即响应请求。worker 被 fork 出的时间是非常短的，
所以不需要花费数秒来等待进程启动。

当这种慢请求发生后，客户端收到 502 错误页面。这个表示请求在处理完成之前被 kill 掉了。

这种处理机制防止了慢请求的产生。

### 内存增长

当某个 worker 使用了过多的内存，外部监控程序如 monit 可以向它发送 QUIT 信号量。
这个将在完成当前的请求后结束掉这个 worker。当 worker 完成请求并结束掉后，
master 进程 fork 出一个新的 worker，并能立即处理请求。
这种方式下我们避免了在请求未完成而被 kill 掉和花费数秒来重启 server。

### 部署

Unicorn 另一精妙之处在于它完成实现零宕机部署，来看它是怎么做的。

首先我们向当前正在服务的 Unicorn master 进程发送一个 USR2 信号量。
这个告诉它开始启动一个新的 master 进程，reloading 所有 app 代码。
新 master 进程完全加载后，它开始 fork 出其需要的 worker 进程。
第一个 worker 发现还有个旧的 master，然后向它发送 QUIT 信号量。

旧 master 进程收到 QUIT 信号量后，它开始优雅的关闭其 workers 。
一旦所有的 workers 完成请求后，旧 master 进程结束。此时我们有了全新的 app 来服务请求，
不需要任何宕机时间。

同样的，可以使用这个方式来更新 Unicorn 它自己。

### 重启

就像上面提到的，重启只会在 master 进程需要重启的时候变慢。
Worker 被 kill 或 fork 的速度是相当快的。

当我们需要重启的时候，只有 master 进程需要重新加载所有 app 代码，
不像其他 server 可能所有进程都需要重新加载所有 app 代码。

## Nginx+Unicorn 配置

首先在 Gemfile 中添加 unicorn:

    gem 'unicorn'

在 config/unicorn.rb 中创建 unicorn 配置。官方 Example 配置文
件在 [github unicorn](https://github.com/defunkt/unicorn/blob/master/examples/unicorn.conf.rb)

在这里可以配置监听端口，如：

    listen 3032, :tcp_nopush => false

配置 worker 进程数量:

    if rails_env == "production"
      worker_processes 6
    else
      worker_processes 1
    end

配置进程ID和日志：

    pid "#{Rails.root}/tmp/pids/unicorn.pid"
    stderr_path "#{Rails.root}/log/unicorn.log"
    stdout_path "#{Rails.root}/log/unicorn.log"

还很多其他的配置可以参考官方文档或 Examle 文件里的注释。

然后启动 unicorn:

    bundle exec unicorn -D -c config/unicorn.rb

现在配置 nginx.conf:

    server {
      listen 80; # for Linux
      server_name slim.bh.com;
      root /home/ryan/ror-apps/nice-slim/public;

      location @app {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;

        proxy_pass http://127.0.0.1:3032;
      }

注意 proxy_pass 设置为 unicorn app 的地址，这样 nginx 就能以 proxy 方式将请求发送到 unicorn。

关于 unicorn.sh 启动脚本，这个脚本方便我们对 unicorn 进行 start, stop,
restart 等操作，这个 unicorn github 本版库也有个 [unicorn.sh](https://github.com/defunkt/unicorn/blob/master/examples/init.sh)
