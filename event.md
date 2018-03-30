#PHP-FPM源码分析 之事件模型

## 事件启动入口

```
fpm\fpm\fpm.c 

int fpm_init(int argc, char **argv, char *config, char *prefix, char *pid, int test_conf, int run_as_root, int force_daemon, int force_stderr) /* {{{ */
{
	fpm_globals.argc = argc;
	fpm_globals.argv = argv;
	if (config && *config) {
		fpm_globals.config = strdup(config);
	}
	fpm_globals.prefix = prefix;
	fpm_globals.pid = pid;
	fpm_globals.run_as_root = run_as_root;
	fpm_globals.force_stderr = force_stderr;

	if (0 > fpm_php_init_main()           ||
	    0 > fpm_stdio_init_main()         ||
	    0 > fpm_conf_init_main(test_conf, force_daemon) ||   //配置信息初始化 事件初始化也在这里
	    0 > fpm_unix_init_main()          ||
	    0 > fpm_scoreboard_init_main()    ||
	    0 > fpm_pctl_init_main()          ||
	    0 > fpm_env_init_main()           ||
	    0 > fpm_signals_init_main()       ||
	    0 > fpm_children_init_main()      ||
	    0 > fpm_sockets_init_main()       ||
	    0 > fpm_worker_pool_init_main()   ||
	    0 > fpm_event_init_main()) {

		if (fpm_globals.test_successful) {
			exit(FPM_EXIT_OK);
		} else {
			zlog(ZLOG_ERROR, "FPM initialization failed");
			return -1;
		}
	}


```
调用顺序是 fpm_conf_init_main->fpm_conf_init_main->fpm_conf_post_process->fpm_event_pre_init

```
fpm\fpm\fpm_conf.c 

static int fpm_conf_post_process(int force_daemon) /* {{{ */
{
...
...

	if (0 > fpm_event_pre_init(fpm_global_config.events_mechanism)) {   //事件初始化 events.mechanism 参数是php-fpm.conf里面的配置参数

		return -1;
	}
...
...
}

```
PS：php-fpm 支持多种事件模型，根据./configure 时来判断支持的事件模型
所有事件模型代码在 fpm/events 下

接着来看一下事件全局的注册


```
fpm\fpm\fpm_conf.c 

//根据宏和配置参数来决定使用的事件模型

int fpm_event_pre_init(char *machanism) /* {{{ */
{
	/* kqueue */
	module = fpm_event_kqueue_module();
	if (module) {
		if (!machanism || strcasecmp(module->name, machanism) == 0) {
			return 0;
		}
	}

	/* port */
	module = fpm_event_port_module();
	if (module) {
		if (!machanism || strcasecmp(module->name, machanism) == 0) {
			return 0;
		}
	}

	/* epoll */
	module = fpm_event_epoll_module();   
	if (module) {
		if (!machanism || strcasecmp(module->name, machanism) == 0) {
			return 0;
		}
	}

	/* /dev/poll */
	module = fpm_event_devpoll_module();
	if (module) {
		if (!machanism || strcasecmp(module->name, machanism) == 0) {
			return 0;
		}
	}

	/* poll */
	module = fpm_event_poll_module();
	if (module) {
		if (!machanism || strcasecmp(module->name, machanism) == 0) {
			return 0;
		}
	}

	/* select */
	module = fpm_event_select_module();
	if (module) {
		if (!machanism || strcasecmp(module->name, machanism) == 0) {
			return 0;
		}
	}

	if (machanism) {
		zlog(ZLOG_ERROR, "event mechanism '%s' is not available on this system", machanism);
	} else {
		zlog(ZLOG_ERROR, "unable to find a suitable event mechanism on this system");
	}
	return -1;
}

```
比如支持epoll的时候
会进入fpm_event_epoll_module函数

```
fpm\fpm\fpm_conf.c 
//这个函数直接返回 结构体 epoll_module
struct fpm_event_module_s *fpm_event_epoll_module() /* {{{ */
{
#if HAVE_EPOLL
	return &epoll_module;
#else
	return NULL;
#endif /* HAVE_EPOLL */
}
/* }}} */

#if HAVE_EPOLL

//还是在本代码文件里，
//epoll_module 结构体定义如下 
static struct fpm_event_module_s epoll_module = {  //fpm_event_module_s结构体是所有事件通用的结构
	.name = "epoll",              //事件模型名字
	.support_edge_trigger = 1,    //没有找到哪里用到
	.init = fpm_event_epoll_init,  //初始化
	.clean = fpm_event_epoll_clean, //清除
	.wait = fpm_event_epoll_wait,   //事件询问
	.add = fpm_event_epoll_add,     //追加事件
	.remove = fpm_event_epoll_remove, //事件移除
};


```

好来看一下事件处理函数


未完待续
