```c

int main(int argc, char *const *argv)
{
    ngx_int_t          i;
    ngx_log_t         *log;
    ngx_cycle_t       *cycle, init_cycle;
    ngx_core_conf_t   *ccf;
    ngx_master_ctx_t   ctx;

#if defined __FreeBSD__
    ngx_debug_init();
#endif

    /* TODO */ ngx_max_sockets = -1;

    ngx_time_init();
// 如果有pcre模块 那么就进行初始化
#if (HAVE_PCRE)
    ngx_regex_init();
#endif
    //获取当前nginx进程的pid
    ngx_pid = ngx_getpid();
//初始化errorlog
    if (!(log = ngx_log_init_stderr())) {
        return 1;
    }

#if (NGX_OPENSSL)
    ngx_ssl_init(log);
#endif

    /* init_cycle->log is required for signal handlers and ngx_getopt() */
    //初始化 ngx_cycle_t
    ngx_memzero(&init_cycle, sizeof(ngx_cycle_t));
    init_cycle.log = log;
    ngx_cycle = &init_cycle;
   //初始化ngx_master_ctx_t,ngx_master_ctx_t 主要存放着 启动参数
    ngx_memzero(&ctx, sizeof(ngx_master_ctx_t));
    ctx.argc = argc;
    ctx.argv = argv;
    //创建pool
    if (!(init_cycle.pool = ngx_create_pool(1024, log))) {
        return 1;
    }
    //获取nginx的参数 -t -c 主要是用来检查配置文件是否正确
    if (ngx_getopt(&ctx, &init_cycle) == NGX_ERROR) {
        return 1;
    }
    //判断如果是检查配置文件
    if (ngx_test_config) {
        log->log_level = NGX_LOG_INFO;
    }
//初始化log
    if (ngx_os_init(log) == NGX_ERROR) {
        return 1;
    }

    if (ngx_add_inherited_sockets(&init_cycle) == NGX_ERROR) {
        return 1;
    }
    ngx_max_module = 0;
    for (i = 0; ngx_modules[i]; i++) {
        ngx_modules[i]->index = ngx_max_module++;
    }
    //初始化init_cycle
    cycle = ngx_init_cycle(&init_cycle);
    if (cycle == NULL) {
        if (ngx_test_config) {
            ngx_log_error(NGX_LOG_EMERG, log, 0,
                          "the configuration file %s test failed",
                          init_cycle.conf_file.data);
        }

        return 1;
    }
    //判断是否是test 如果是校验配置文件的话就直接返回
    if (ngx_test_config) {
        ngx_log_error(NGX_LOG_INFO, log, 0,
                      "the configuration file %s was tested successfully",
                      init_cycle.conf_file.data);
        return 0;
    }

    ngx_os_status(cycle->log);

    ngx_cycle = cycle;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    ngx_process = ccf->master ? NGX_PROCESS_MASTER : NGX_PROCESS_SINGLE;

#if (WIN32)

#if 0

    TODO:

    if (ccf->run_as_service) {
        if (ngx_service(cycle->log) == NGX_ERROR) {
            return 1;
        }

        return 0;
    }
#endif

#else

    if (!ngx_inherited && ccf->daemon) {
        if (ngx_daemon(cycle->log) == NGX_ERROR) {
            return 1;
        }

        ngx_daemonized = 1;
    }

    if (ngx_create_pidfile(cycle, NULL) == NGX_ERROR) {
        return 1;
    }

#endif
//  判断如果是master 模式 就是启动master和worker进程，否则就是启用单进程模式
    if (ngx_process == NGX_PROCESS_MASTER) {
        ngx_master_process_cycle(cycle, &ctx);

    } else {
        ngx_single_process_cycle(cycle, &ctx);
    }

    return 0;
}


```