---
layout: post
title: 用Nginx做正向代理时获取其向外连接的IP和端口
description: "获取向外连接的IP和端口"
category: 技术
tags: [正向代理, Nginx, IP, 端口]
---
{% include JB/setup %}

有时候你可能需要知道Nginx向外连接时用了哪个端口，这些端口是当connect(ngx_event_connect.c - ngx_event_connect_peer(ngx_peer_connection_t *pc))时是由系统自动分配的。应用场景是这样的：

[Client]->[Nginx]->[Internet]

Nginx作为正向代理，我们需要获取的时它向外连接时的IP和端口，IP很好获取，可从Nginx系统变量里面获得，而且不会改变，但是端口是有系统分配的，Nginx本身也没有现成的变量可用。

到这里你可能首先想到的是，为什么要获取这个端口，这个端口信息基本没有意义，也许它可以作为回溯系统分析中数据的一部分。

我们将设置一个新的变量$upstream_laddr, 日志中可以用它打印出来。这个变量在ngx_http_upstram.c 中设置，其ngx_http_variable_t结构中的get_handler函数为：

		static ngx_int_t ngx_http_upstream_laddr_variable(ngx_http_request_t *r, ngx_http_variable_value_t *v, uintptr_t data) {

		ngx_peer_connection_t *pc; 
		struct sockaddr_in *sin; 
		u_char sa[NGX_SOCKADDRLEN]; 
		socklen_t len; 
		#if (NGX_HAVE_INET6) 
		ngx_uint_t i; 
		struct sockaddr_in6 *sin6; 
		#endif

		ngx_str_t s; 
		u_char addr[NGX_SOCKADDR_STRLEN];

		s.len = NGX_SOCKADDR_STRLEN; 
		s.data = addr;

		if (r->upstream == NULL) { return NGX_ERROR; }

		pc = &r->upstream->peer; 
		if (pc == NULL){ return NGX_ERROR; } 
		if (pc->local_sockaddr == NULL) { return NGX_ERROR;} /* 需要去ngx_peer_connection_t 结构中添加local_sockaddr变量 */

		s.len = ngx_sock_ntop(pc->local_sockaddr, s.data, s.len, 1);

		s.data = ngx_pnalloc(r->pool, s.len); if (s.data == NULL) { return NGX_ERROR; }

		ngx_memcpy(s.data, addr, s.len);

		v->len = s.len; 
		v->valid = 1; 
		v->no_cacheable = 0; 
		v->not_found = 0; 
		v->data = s.data;

		return NGX_OK; 
		} 

同时，在 ngx_event_connect_peer(ngx_peer_connection_t *pc) 连接后要通过getsockname函数获得IP信息，并存储在local_sockaddr变量中，像这样：

	if (getsockname(c->fd, (struct sockaddr *) &sa, &len) == -1) {
    		ngx_connection_error(c, ngx_socket_errno, "getsockname() failed");
        	return NGX_ERROR;
    	}

    	c->local_sockaddr = ngx_palloc(c->pool, len);
    	if (c->local_sockaddr == NULL) {
        	return NGX_ERROR;
    	}

    	ngx_memcpy(c->local_sockaddr, &sa, len);
    
这段代码你需要加入在ngx_event_connect_peer执行之后，为了了解Nginx的upstream 执行的顺利，下面有一张不太严谨的流程图可以参考：

所以你可以加在ngx_http_upstream_connect执行之后。最后你可以直接在配置文件中使用变量$upstream_laddr了。
