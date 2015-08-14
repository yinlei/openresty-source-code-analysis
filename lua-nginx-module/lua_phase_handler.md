#Phase Handler

## Lua 在请求中

Lua 在初始化过程中的调用咱们都有大致的了解了，但仅仅是这些只是，我们对 Lua 在模块中执行的理解还不够，接下来咱们来深入了解请求到来时的各个 Phase 被插入的 handler：

## rewrite_by_lua

指令定义：

```c
    { ngx_string("rewrite_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_rewrite_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_rewrite_handler_inline },
```
能看出来 rewrite_by_lua 跟 init_by_lua 不一样的地方是，允许在 main ，server ， location ， if 上下文中使用，可以看出 rewrite_by_lua 的使用将会比 init_by_lua* 更加灵活。
从 ngx_http_lua_init 函数代码分析我们可以得知， rewrite_by_lua 指令定义的 Lua 代码，首先是把 ngx_http_lua_rewrite_handler 插入到 cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers 列表中，在一个请求处理的 rewrite 阶段被执行。
那么被插入的 ngx_http_lua_rewrite_handler 函数具体的实现了什么：

```c

ngx_int_t
ngx_http_lua_rewrite_handler(ngx_http_request_t *r)
{
    ngx_http_lua_loc_conf_t     *llcf;
    ngx_http_lua_ctx_t          *ctx;
    ngx_int_t                    rc;
    ngx_http_lua_main_conf_t    *lmcf;

    // 好像已经有跳转了，往下一个模块抛
    if (r->uri_changed) {
        return NGX_DECLINED;
    }

    lmcf = ngx_http_get_module_main_conf(r, ngx_http_lua_module);

    if (!lmcf->postponed_to_rewrite_phase_end) {
    
        // 这个操作整个 worker 进程只操作一次，把自己的 handler 放到了 rewrite 阶段最后的位置
        ngx_http_core_main_conf_t       *cmcf;
        ngx_http_phase_handler_t        tmp;
        ngx_http_phase_handler_t        *ph;
        ngx_http_phase_handler_t        *cur_ph;
        ngx_http_phase_handler_t        *last_ph;

        lmcf->postponed_to_rewrite_phase_end = 1;
        cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

        ph = cmcf->phase_engine.handlers;
        cur_ph = &ph[r->phase_handler];
        last_ph = &ph[cur_ph->next - 1];

        if (cur_ph < last_ph) {
            tmp      = *cur_ph;
            memmove(cur_ph, cur_ph + 1,
                (last_ph - cur_ph) * sizeof (ngx_http_phase_handler_t));

            *last_ph = tmp;

            // 重新跑一次 当前序号的 handler 
            r->phase_handler--; /* redo the current ph */
            return NGX_DECLINED;
        }
    }

    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);

    // 没有配置 rewrite_by_lua* 的 Lua 代码，跳到下一 handler
    if (llcf->rewrite_handler == NULL) {
        dd("no rewrite handler found");
        return NGX_DECLINED;
    }

    // 要初始化一个 ctx ， ngx_http_lua_create_ctx 函数这里内容还挺多 
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        ctx = ngx_http_lua_create_ctx(r);
        if (ctx == NULL) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
    }

    if (ctx->entered_rewrite_phase) {
    
        // 在执行完 rewrite_by_lua* 的代码后，又没有跳到下个 handler ，回到这里才会执行 
        rc = ctx->resume_handler(r);
        if (rc == NGX_OK) {
            rc = NGX_DECLINED;
        }

        if (rc == NGX_DECLINED) {
            if (r->header_sent) {
                if (!ctx->eof) {
                    rc = ngx_http_lua_send_chain_link(r, ctx, NULL);
                    if (rc == NGX_ERROR || rc > NGX_OK) {
                        return rc;
                    }
                }

                return NGX_HTTP_OK;
            }
        }

        return rc;
    }

    // waiting_more_body 被设置时，直接结束 request 处理
    if (ctx->waiting_more_body) {
        return NGX_DONE;
    }

    // 响应指令 lua_need_request_body 设置，读取 body 数据，获得数据后再重新运行这个 handler 
    if (llcf->force_read_body && !ctx->read_body_done) {
        r->request_body_in_single_buf = 1;
        r->request_body_in_persistent_file = 1;
        r->request_body_in_clean_file = 1;

        rc = ngx_http_read_client_request_body(r,
                                       ngx_http_lua_generic_phase_post_read);

        if (rc == NGX_AGAIN) {
            // 这里设置了 waiting_more_body
            ctx->waiting_more_body = 1;
            return NGX_DONE;
        }
    }

    // 终于来到这里，跑 Lua 代码了
    return llcf->rewrite_handler(r);
}

```

这个函数花了很多的代码处理了各种逻辑，关键代码调用 Lua 的地方，就是最后一条语句，所调用函数指针就是之前通过 ngx_http_lua_rewrite_by_lua 函数传入的 ngx_http_lua_rewrite_handler_inline 函数。

```c

ngx_int_t
ngx_http_lua_rewrite_handler_inline(ngx_http_request_t *r)
{
    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);
    L = ngx_http_lua_get_lua_vm(r, NULL);

    /*  load Lua inline script (w/ cache) sp = 1 */
    rc = ngx_http_lua_cache_loadbuffer(r, L, llcf->rewrite_src.value.data,
                                       llcf->rewrite_src.value.len,
                                       llcf->rewrite_src_key,
                                       (const char *)
                                       llcf->rewrite_chunkname);
    if (rc != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    return ngx_http_lua_rewrite_by_chunk(L, r);
}

static ngx_inline lua_State *
ngx_http_lua_get_lua_vm(ngx_http_request_t *r, ngx_http_lua_ctx_t *ctx)
{
    // 从模块的上下文中获取状态机
    if (ctx == NULL) {
        ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    }

    if (ctx && ctx->vm_state) {
        return ctx->vm_state->vm;
    }

    // 从模块的配置中获取状态机
    lmcf = ngx_http_get_module_main_conf(r, ngx_http_lua_module);
    return lmcf->lua;
}

ngx_int_t
ngx_http_lua_cache_loadbuffer(ngx_http_request_t *r, lua_State *L,
    const u_char *src, size_t src_len, const u_char *cache_key,
    const char *name)
{
    // 根据 cache_key 加载 Lua 代码到虚拟机上
    rc = ngx_http_lua_cache_load_code(r, L, (char *) cache_key);
    if (rc == NGX_OK) {
        return NGX_OK;
    }

    // cache_key 找不到，老老实实加载 Lua 代码，这里要注意“代码工厂”的概念
    rc = ngx_http_lua_clfactory_loadbuffer(L, (char *) src, src_len, name);
    
    // 把刚加载的 Lua 代码块按 cache_key 索引存储起来 
    rc = ngx_http_lua_cache_store_code(L, (char *) cache_key);

    return NGX_OK;

}

```

ngx_http_lua_rewrite_handler_inline 函数从 ngx_http_lua_get_lua_vm 获取 Lua 虚拟机，并通过 ngx_http_lua_cache_loadbuffer 获取 Lua 代码，然后调用 ngx_http_lua_rewrite_by_chunk 执行 Lua 代码。这里关键的一点也在最后一句，这个 handler 的最后会把 Lua 执行代码的结果作为返回，并决定下一个处理阶段应该交给哪个 handler 处理。

这里有一个疑问， llcf->rewrite_src_key (也就是 cache_key ) 从哪儿来的？其实 cache_key 是在 ngx_http_lua_rewrite_by_lua 指令处理函数计算出来的。

咱们回头看看， 配置读取阶段 ngx_http_lua_rewrite_by_lua 函数做了什么。

```c

char *
ngx_http_lua_rewrite_by_lua(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    if (cmd->post == ngx_http_lua_rewrite_handler_inline) {
        chunkname = ngx_http_lua_gen_chunk_name(cf, "rewrite_by_lua",
                                                sizeof("rewrite_by_lua") - 1);

        llcf->rewrite_chunkname = chunkname;
        llcf->rewrite_src.value = value[1];

        p = ngx_palloc(cf->pool, NGX_HTTP_LUA_INLINE_KEY_LEN + 1);
        llcf->rewrite_src_key = p;

        // llcf->rewrite_src_key 就是这里计算出来的
        p = ngx_copy(p, NGX_HTTP_LUA_INLINE_TAG, NGX_HTTP_LUA_INLINE_TAG_LEN);
        p = ngx_http_lua_digest_hex(p, value[1].data, value[1].len);
        *p = '\0';

    } else {
        ngx_memzero(&ccv, sizeof(ngx_http_compile_complex_value_t));
        ccv.cf = cf;
        ccv.value = &value[1];
        ccv.complex_value = &llcf->rewrite_src;

        // 读取 Lua 文件情况下，如果文件名是个变量，llcf->rewrite_src_key 就按如下方式计算了
        if (ngx_http_compile_complex_value(&ccv) != NGX_OK) {
            return NGX_CONF_ERROR;
        }

        // 如果不是变量，上面的计算不出来，就把 Lua 文件名用来计算了
        if (llcf->rewrite_src.lengths == NULL) {

            p = ngx_palloc(cf->pool, NGX_HTTP_LUA_FILE_KEY_LEN + 1);
            llcf->rewrite_src_key = p;

            p = ngx_copy(p, NGX_HTTP_LUA_FILE_TAG, NGX_HTTP_LUA_FILE_TAG_LEN);
            p = ngx_http_lua_digest_hex(p, value[1].data, value[1].len);
            *p = '\0';
        }
    }

    // 这里把 ngx_http_lua_rewrite_handler_inline 作为 rewrite_handler 设置了
    llcf->rewrite_handler = (ngx_http_handler_pt) cmd->post;

    lmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_lua_module);

    // 这里引导 ngx_http_lua_init 挂载 handler 进 rewrite 阶段列表
    lmcf->requires_rewrite = 1;
    
    // 这里引导 ngx_http_lua_init 挂载 capture_filter 进 output filter中
    lmcf->requires_capture_filter = 1;

    return NGX_CONF_OK;
}
```

ngx_http_lua_rewrite_by_lua 函数，在处理 rewrite_by_lua* 指令时在 location 的配置结构上设置了 rewrite_handler 的回调，计算了 Lua 代码的 cache_key，然后在请求处理的 rewrite 阶段，通过这些回调处理指令输入的 Lua 代码，执行结果对 rewrite 阶段产生影响。

## access_by_lua 系列
跟 rewrite_by_lua\* 指令类似的还有， access_by_lua\*  和 content_by_lua\*, log_by_lua\*，他们的区别是插入的阶段不一样，我们除了要考虑编写正确的 Lua 代码的同时，还要掌握好根据 Lua 代码功能，在恰当的请求处理阶段插入 Lua 代码。
一般情况下： 

*  rewrite_by_lua* 上的 Lua 代码用于处理跳转逻辑。
*  access_by_lua* 上的 Lua 代码用于访问权限逻辑。
*  content_by_lua* 上的 Lua 代码用于内容生成逻辑。
*  log_by_lua* 上的 Lua 代码用于写日志之后做剩余收尾逻辑，比如说统计或者其他的。

access_by_lua*，这个阶段大部分与 rewrite_by_lua*类似，大致流程是这样的：

配置读取阶段(ngx_http_lua_access_by_lua)： 
*  根据 Lua 代码字符串计算 cache_key
*  向 location 配置挂载 access_handler（也就是 ngx_http_lua_access_handler_inline ）

初始化阶段(ngx_http_lua_init)：
*  向 core 模块 main 配置的 access 阶段处理回调队列插入 ngx_http_lua_access_handler

请求处理的 access 阶段( ngx_http_lua_access_handler )：
*  一次性地，把 ngx_http_lua_access_handler 放到 access 阶段处理回调队列的最后，也就是说明文档所说的，跟在 HttpAccessModule 的处理过后执行。
*  调用 access_handler 回调，也就是 ngx_http_lua_access_handler_inline 
*  ngx_http_lua_access_handler_inline 内，获取 Lua 虚拟机
*  通过 cache_key 获取代码块，并在 Lua 虚拟机中运行这个代码块。

content_by_lua\* 有一点点区别：

*  配置读取阶段( ngx_http_lua_content_by_lua )，向 location 的 ngx_http_core_module 模块设置 handler 回调为 ngx_http_lua_content_handler
*  初始化阶段( ngx_http_lua_init )，没有像其他指令一样，它并没有向某个阶段插入回调。
*  请求处理的 content 阶段( ngx_http_lua_content_handler )，也没有像其他指令一样一次性的挪动 handler 操作，因为 content handler 只能有一个。

## *_filter_by_lua 系列
header_filter_by_lua\* 和 body_filter_by_lua\* 这两个指令允许在 发送 header 以及 body 前，执行 Lua 代码对将要发送的内容进行过滤。

在一般处理流程中， content 阶段的 hanlder 需要执行 ngx_http_send_header 以及 ngx_http_output_filter 以分别把 header 以及 body 作为应答数据发送给客户端。
ngx_http_send_header 函数会调用 ngx_http_top_header_filter，各种 filter 模块要实现其 filter 功能，就必须 把 ngx_http_top_header_filter 指针设定成自己模块的一个函数，然后把原本的 ngx_http_top_header_filter 保存在本模块的 ngx_http_next_header_filter 上，并在 自己的 filter 函数中调用 ngx_http_next_header_filter ，这样子居然形成了一个回调链条， filter 被准确无误的挨个被执行了。

filter 模块可以实现的功能挺多，可以对 content 阶段产生的内容做各种加工处理，比如说， nginx 内部的 gzip 过滤模块，就利用 filter 特性把内容从明文转成 gzip。而他的初始化代码如下：

```c

static ngx_http_output_header_filter_pt  ngx_http_next_header_filter;
static ngx_http_output_body_filter_pt    ngx_http_next_body_filter;

static ngx_int_t
ngx_http_gzip_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_gzip_header_filter;

    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter = ngx_http_gzip_body_filter;

    return NGX_OK;
}

```

基本上 filter 模块的初始化都要做把模块的 filter 函数插入到 header 和 body filter 链表头上，所以越晚注册的模块越早执行。按 ngx_modules 模块列表定义，header 的最后一个可执行 filter 是 ngx_http_header_filter_module ， 而 body 的最后一个可执行 filter 是 ngx_http_write_filter_module ，他们的指责就是往客户端发送最终的数据。

我们的主角 lua-nginx-module 模块也不例外，被同样被定义在这两个模块之后，所以他们会可以在最终发送给用户前，内容先被 lua-nginx-module 处理一遍。

读取配置阶段与之前的指令处理流程基本类似，在 header 的 filter 处理如下：

```c

static ngx_int_t
ngx_http_lua_header_filter(ngx_http_request_t *r)
{
    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);

    if (llcf->header_filter_handler == NULL) {
        // 如果 location 没指定 Lua 代码，跳过本 filter
        return ngx_http_next_header_filter(r);
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    old_context = ctx->context;
    ctx->context = NGX_HTTP_LUA_CONTEXT_HEADER_FILTER;

    // 调用 filter 的 Lua 代码 
    rc = llcf->header_filter_handler(r);

    ctx->context = old_context;

    if (rc == NGX_DECLINED) {
        return NGX_OK;
    }

    if (rc == NGX_AGAIN || rc == NGX_ERROR) {
        return rc;
    }

    return ngx_http_next_header_filter(r);
}

```

