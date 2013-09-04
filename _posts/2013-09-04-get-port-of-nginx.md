---
layout: post
title: ��Nginx���������ʱ��ȡ���������ӵ�IP�Ͷ˿�
description: "��ȡ�������ӵ�IP�Ͷ˿�"
category: ����
tags: [�������, Nginx, IP, �˿�]
---
{% include JB/setup %}

��ʱ���������Ҫ֪��Nginx��������ʱ�����ĸ��˿ڣ���Щ�˿��ǵ�connect(ngx_event_connect.c - ngx_event_connect_peer(ngx_peer_connection_t *pc))ʱ����ϵͳ�Զ�����ġ�Ӧ�ó����������ģ�

[Client]->[Nginx]->[Internet]

Nginx��Ϊ�������������Ҫ��ȡ��ʱ����������ʱ��IP�Ͷ˿ڣ�IP�ܺû�ȡ���ɴ�Nginxϵͳ���������ã����Ҳ���ı䣬���Ƕ˿�����ϵͳ����ģ�Nginx����Ҳû���ֳɵı������á�

����������������뵽���ǣ�ΪʲôҪ��ȡ����˿ڣ�����˿���Ϣ����û�����壬Ҳ����������Ϊ����ϵͳ���������ݵ�һ���֡�

���ǽ�����һ���µı���$upstream_laddr, ��־�п���������ӡ���������������ngx_http_upstram.c �����ã���ngx_http_variable_t�ṹ�е�get_handler����Ϊ��

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
		if (pc->local_sockaddr == NULL) { return NGX_ERROR;} /* ��Ҫȥngx_peer_connection_t �ṹ�����local_sockaddr���� */

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

ͬʱ���� ngx_event_connect_peer(ngx_peer_connection_t *pc) ���Ӻ�Ҫͨ��getsockname�������IP��Ϣ�����洢��local_sockaddr�����У���������

		if (getsockname(c->fd, (struct sockaddr *) &sa, &len) == -1) {
    		ngx_connection_error(c, ngx_socket_errno, "getsockname() failed");
        return NGX_ERROR;
    }

    c->local_sockaddr = ngx_palloc(c->pool, len);
    if (c->local_sockaddr == NULL) {
        return NGX_ERROR;
    }

    ngx_memcpy(c->local_sockaddr, &sa, len);
    
��δ�������Ҫ������ngx_event_connect_peerִ��֮��Ϊ���˽�Nginx��upstream ִ�е�˳����������һ�Ų�̫�Ͻ�������ͼ���Բο���

��������Լ���ngx_http_upstream_connectִ��֮����������ֱ���������ļ���ʹ�ñ���$upstream_laddr�ˡ�