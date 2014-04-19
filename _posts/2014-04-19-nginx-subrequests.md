---
layout: post
title: 用Lua脚本编写Nginx的subrequest
description: "用Lua脚本编写Nginx的subrequest"
category: 技术
tags: [Nginx, Lua, Subrequest]
---
{% include JB/setup %}

在某些应用场景中，nginx收到一个请求后，需要向多个后端发送子请求，当所有子请求都返回后还需要对这些返回结果进行过滤再发送给客户端。对于这样的需要我是采用编写handler模块和filter模块来实现的，但是用C编写模块的时间相对较长而且调试的时候比较痛苦（异步的原因）。所以最近就想着是否用Lua来写，并查找到相关资料，发现还真是方便，Lua的协程将nginx中并发异步处理透明化，让你觉得是在写串行程序，两个字形容：”真爽“。

用Lua重新编写完原来的程序后，发现代码行数少了很多，Lua写总共才160多行，而以前的C代码大约有1500行左右。虽然Lua没有学过，但是进行的比较顺利（以前从来没学过，只是知道），因为有丰富的官方文档。可是在调试时却发现怎么也无法有效发出HTTP POST子请求（数据比较大，>80k <100k左右），经过仔细的调试代码，并查阅相关文档，才发现是没有有效的发送出request body的缘故。

这里先贴出Lua脚本源码，首先是nginx.conf中的配置：

		location / {
    	content_by_lua_file  /usr/local/nginx/conf/subrequest.lua; # 加载lua脚本文件
    }
    location /sub2 { # 发出的子请求location
      rewrite ^/sub2(.*)$ $1 break; # 重写URL，主要是去掉sub2，并保留其他的参数信息，注意使用break不会照成多次内部重定向
      proxy_pass http://192.168.1.1:12345/;
    }
    location /sub1 { # 发出的子请求location
      rewrite ^/sub1(.*)$ $1 break;
      proxy_pass http://192.168.1.2:12345/;
    }

    location /sub3 { # 发出的子请求location
      rewrite ^/sub3(.*)$ $1 break;
      proxy_pass http://192.168.1.3:12345/;
    }

然后是subrequest.lua脚本：

    local action = ngx.var.request_method;
		-- important very much!  #这里非常重要，一开始我没有添加这行，导致没有成功发出子请求
		ngx.req.read_body();

		-- get request body
		local data = ngx.req.get_body_data();
		-- get request uri's args
		local args = ngx.req.get_uri_args();

		if action == "POST" then
    	arry = {method=ngx.HTTP_POST, body=data};
		else
    	arry = {method=ngx.HTTP_GET};
		end

		-- issue subrequest.
		local res1,res2,res3 = ngx.location.capture_multi({
    	{"/sub1"..ngx.var.request_uri, {method = ngx.HTTP_POST, body=data}},
    	{"/sub2"..ngx.var.request_uri, {method = ngx.HTTP_POST, body=data}},
    	{"/sub3"..ngx.var.request_uri, {method = ngx.HTTP_POST, body=data}}
    	-- TODO: N's subrequests
		})

		-- 对返回结果进行业务处理
		if res1.status == ngx.HTTP_OK then
    	local body = res1.body;
    ...................................
    ...................................

为了能让Nginx执行上面的代码，你需要按照这里的说明进行安装，或者你直接安装最新的openresty也是可以的。上面有很多nginx lua API的调用，你可以通过查阅官方的文档进行了解，比如ngx.location.capture_multi是发起多个子请求、获取URL参数ngx.req.get_uri_args，返回是Lua的table类型。是不是很简单？你也试试吧。


