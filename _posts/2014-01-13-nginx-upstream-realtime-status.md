---
layout: post
title: Nginx upstream 实时状态监控
description: "Nginx upstream 实时状态监控"
category: 技术
tags: [实时, 监控, Nginx模块]
---
{% include JB/setup %}

Nginx本身自带了stubs_status状态监控模块，并且默认是不被加入编译的，需要用它时可以在配置中(./configure命令选项)添加--with-http_stub_status_module选项。这个状态监控模块非常有用，能提供Nginx的实时请求状态，但却不知道为什么没有被添加到Nginx的默认模块中，我想应该是考虑到它会额外的增加Nginx的性能消耗，并且监控手段也并不仅此一种，而且Nginx的这个状态监控模块是比较简单的，它的使用和指南可以参考[官方指南](http://wiki.nginx.org/HttpStubStatusModule)，这里有较为详细的说明，并且在页面底部还特意标出了可选的其他监控Nginx的第三方解决方案，如Collectd。

在我的工作环境中，需要实时的查看backends的响应时间、HTTP返回状态统计、发送请求数等统计，一开始我希望能通过像RRDTool这样的工具来完成，但实时上它是依赖于stub_status的输出的，而stub_status并不能提供我想要的信息，虽然我可以在日志里面打印出所有我想要的数据，但这意味我要对日志文件进行“监控”，想来想去，就决定自己去写一个监控upstream的模块。

先一睹为快，看看最终实现的返回结果：

![图1 返回结果]({{ site.img_url }}/stubs-status.png)

部分代码解析如下：

设置回调函数，放在log模块之后
		static ngx_int_t
		ngx_http_stubs_status_handler_init(ngx_conf_t \*cf)
		{
    		ngx_http_handler_pt         \*h;
    		ngx_http_core_main_conf_t   \*cmcf;

    		cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    		h = ngx_array_push(&cmcf->phases[NGX_HTTP_LOG_PHASE].handlers);
    		if (h == NULL) {
        	return NGX_ERROR;
    		}
    		/* set callback, after log phase */
    		\*h = ngx_http_stubs_status_request_handler;

    		return NGX_OK;
		}

这个函数已经被设置成回调了，它负责计算并记录输出变量的值

		static ngx_int_t
		ngx_http_stubs_status_request_handler(ngx_http_request_t \*r)
		{
    		ngx_http_stubs_status_conf_t        \*sscf;
    		ngx_http_stubs_status_ctx_t         \*ctx;

    		ngx_http_upstream_state_t           \*state;
    		size_t          response_length = 0;
    		ngx_atomic_t    response_time = 0;

    		ngx_uint_t      i;
    		ngx_msec_int_t  ms;

    		sscf = ngx_http_get_module_main_conf(r, ngx_http_stubs_status_module);
    		ctx = sscf->ctx;


    		if (NULL == sscf || NULL == ctx) {
        		return NGX_ERROR;
    		}

    		if (r->upstream_states != NULL)
    		{
        		i = 0;
        		ms = 0;
        		state = r->upstream_states->elts;

        		for ( ;; ) {           
            		if (state[i].status) {
                		ms = (ngx_msec_int_t)
                        (state[i].response_sec * 1000 + state[i].response_msec);
                		ms = ngx_max(ms, 0);
            		}

            		response_length += state[i].response_length;
            		response_time += (ngx_atomic_t) ms;
            
            		if (++i == r->upstream_states->nelts) {
                		break;
            		}
            
            		//ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
            		//    "state_response_time[%d]: \"%d\"", i, state[i].response_msec);
        		}

        		ngx_atomic_fetch_add(&ctx->stat_response_recv, response_length);
        		ngx_atomic_fetch_add(&ctx->stat_response_time_1m, response_time);

        		ngx_atomic_fetch_add(&ctx->stat_requests, 1);
        		ngx_atomic_fetch_add(&ctx->stat_sent, r->connection->sent);    
    
        		/* per and avg */    
        		ngx_atomic_fetch_add(&ctx->stat_requests_1m, 1);

        		ngx_time_t  *time = ngx_timeofday();

        		ngx_shmtx_lock(&sscf->shpool->mutex);
        		if (time->sec - ctx->stat_time >= 60) {
            		if (0 == ctx->stat_requests_1m) {
                		ctx->stat_per_requests = 0;
                		ctx->stat_avg_response_time = 0;
            		} else if (0 == ctx->stat_response_time_1m) {
                		ctx->stat_avg_response_time = 0;
            		} else {
                		ctx->stat_per_requests = ctx->stat_requests_1m / 60;
                		ctx->stat_avg_response_time = ctx->stat_response_time_1m 
                    	/ ctx->stat_requests_1m;
                		//ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                		//            "avg_response_time: \"%d\"", ctx->stat_avg_response_time);
            		}
            		time = ngx_timeofday();

            		ctx->stat_time = time->sec;
            		ctx->stat_requests_1m = 0;
            		ctx->stat_response_time_1m = 0;
        		}
        		ngx_shmtx_unlock(&sscf->shpool->mutex);

        		/* stat HTTP status */
        		if (r->err_status) {
            		if (r->err_status >= 400 && r->err_status < 410)
            		{
                		ngx_atomic_fetch_add(&ctx->stat_request_40x, 1);
            		}else if (r->err_status >= 500 && r->err_status < 510)
            		{
                		ngx_atomic_fetch_add(&ctx->stat_request_50x, 1);
            		}
        		} else if (r->headers_out.status) {
            		if (r->headers_out.status >= 200 && r->headers_out.status < 210)
            		{
                		ngx_atomic_fetch_add(&ctx->stat_request_20x, 1);
            		}else if (r->headers_out.status >= 300 && r->headers_out.status < 310)
            		{
                		ngx_atomic_fetch_add(&ctx->stat_request_30x, 1);
            		}
        		}        
    		} // if r->upstream_states != NULL
       
    		return NGX_OK; 
		}

我们还需要一个HTTP处理函数，当有来自某个location的请求时调用该函数

		static ngx_int_t
		ngx_http_stubs_status_handler(ngx_http_request_t \*r)
		{
    		size_t      size;
    		ngx_int_t   rc;
    		ngx_buf_t  \*b;
    		ngx_chain_t out;

   			ngx_time_t *time;    

    		ngx_http_stubs_status_conf_t \*sscf;
    		ngx_http_stubs_status_ctx_t  \*ctx;

    		ngx_atomic_int_t rq, st, rv, pr, at;
    		ngx_atomic_int_t r2, r3, r4, r5;

    		if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD))) {
        		return NGX_HTTP_NOT_ALLOWED;
    		}

    		if (r->uri.data[r->uri.len - 1] == '/') {
        		return NGX_DECLINED;
    		}

    		rc = ngx_http_discard_request_body(r);

    		if (rc != NGX_OK) {
        		return rc;
    		}

   	 		/* set response content type */
    		ngx_str_set(&r->headers_out.content_type, "text/plain");    
    
    		if (r->method == NGX_HTTP_HEAD) {
        		r->headers_out.status = NGX_HTTP_OK;

        		rc = ngx_http_send_header(r);

        		if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
            		return rc;
        		}
    		}

    		sscf = ngx_http_get_module_main_conf(r, ngx_http_stubs_status_module);
    		if (NULL == sscf) {
        		return NGX_ERROR;
    		}

    		size = sizeof("Uptime: \n") + NGX_ATOMIC_T_LEN
         	+ sizeof("upstream requests: \n")
         	+ sizeof("upstream sent: \n")
         	+ sizeof("upstream recv: \n")
         	+ sizeof("upstream reqs/per: \n")
         	+ sizeof("upstream resp_time/avg(ms): \n")
         	+ sizeof("-------------------------------------\n")
         	+ 8 + 5 * NGX_ATOMIC_T_LEN
         	+ sizeof("reqs_20x: \n") 
         	+ sizeof("reqs_30x: \n")
         	+ sizeof("reqs_40x: \n")
         	+ sizeof("reqs_50x: \n")
         	+ 4 * NGX_ATOMIC_T_LEN;

    		/* create buf */
    		b = ngx_create_temp_buf(r->pool, size);
    		if (NULL == b) {
        		return NGX_HTTP_INTERNAL_SERVER_ERROR;
    		}

    		out.buf = b;
    		out.next = NULL;

    		/* fill stat data into buf */
    		ctx = sscf->ctx;
    		rq = ctx->stat_requests;
    		st = ctx->stat_sent;
    		rv = ctx->stat_response_recv;
    		pr = ctx->stat_per_requests;
    		at = ctx->stat_avg_response_time;
    		r2 = ctx->stat_request_20x;
    		r3 = ctx->stat_request_30x;
    		r4 = ctx->stat_request_40x;
    		r5 = ctx->stat_request_50x; 

    		time = ngx_timeofday();     
    		b->last = ngx_sprintf(b->last, "Uptime: %uA\n", time->sec - sscf->startup);
    
    		b->last = ngx_sprintf(b->last, "upstream requests: %uA\n", rq);    
    		b->last = ngx_sprintf(b->last, "upstream sent: %uA\n", st);    
    		b->last = ngx_sprintf(b->last, "upstream recv: %uA\n", rv);    
    		b->last = ngx_sprintf(b->last, "upstream reqs/per: %uA\n", pr);    
    		b->last = ngx_sprintf(b->last, "upstream resp_time/avg(ms): %uA\n", at);    

    		b->last = ngx_cpymem(b->last, "-------------------------------------\n",
        		sizeof("-------------------------------------\n") - 1);

    		b->last = ngx_sprintf(b->last, "reqs_20x: %uA\n", r2);     
    		b->last = ngx_sprintf(b->last, "reqs_30x: %uA\n", r3);     
    		b->last = ngx_sprintf(b->last, "reqs_40x: %uA\n", r4);     
    		b->last = ngx_sprintf(b->last, "reqs_50x: %uA\n", r5);    


    		r->headers_out.status = NGX_HTTP_OK;
    		r->headers_out.content_length_n = b->last - b->pos;

    		b->last_buf = (r == r->main) ? 1 : 0;

    		rc = ngx_http_send_header(r);

    		if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        		return rc;
    		}

    		return ngx_http_output_filter(r, &out);
		}

为了让该模块在最后被执行(这样才能保证你获取的结果是最终结果，因为异步HTTP的缘故，事件会多次到达)，还需要在配置中做些小处理：

		ngx_addon_name=ngx_http_stubs_status_module
		# Make module to last execute
		HTTP_MODULES2="$HTTP_MODULES"
		HTTP_MODULES=" $HTTP_MODULES2 ngx_http_stubs_status_module"

		NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_stubs_status_module.c"

好了，目前该模块还存在不少问题，如：当前是按照每分钟计算平均请求数和平均延时，另外还不支持多个server(vhost)的监控，我会在以后添加上来。

源码可从[github](https://github.com/honwel)获取，如果你觉得该模块有用，并且还在使用过程中发现了问题，那么你可以通过邮件把你遇到的问题或者代码中的错误做个简单的描述，然后发送给我，非常感谢。
