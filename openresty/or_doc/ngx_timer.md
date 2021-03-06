# ngx.timer.at

*语法：* `ok, err = ngx.timer.at(delay, callback, user_arg1, user_arg2, ...)`
*上下文：* `init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.*, balancer_by_lua*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*, ssl_session_store_by_lua*`

*使用一个自定义函数以及可选的自定义参数创建一个 Nginx 计时器*

第一个参数 `delay` 以秒为单位指定计时器的延迟时间，支持分秒设置，比如 0.001 在这里表示 1 毫秒延迟。`delay` 同样可以设置为 0 ，此时如果当前句柄正被唤醒则计时器将立即获得执行。（in which case the timer will immediately expire when the current handler yields execution.//TODO yields）

第二个参数 `callback` 可以是任何 Lua 函数，后期延迟时间到了，该函数将被以一个后台 “轻线程” 的形式被调用。这个自定义的回调函数将被 Nginx 核心使用 `premature` 参数、user_arg1、user_arg2 等参数自动调用，参数 `premature` 是一个 boolean 值，表示当前定时器是否过期以后，而 user_arg1、user_arg2 等参数就是调用 `ngx.timer.at` 时所传递的余下参数列表。

当 Nginx 工作进程尝试关闭，比如在 Nginx 由于收到 HUP 信号而触发了 Nginx 配置重载的时候，或者 Nginx 服务正在关闭的时候，将会出现无效的计时器（//TODO Premature timer）。当 Nginx 工作进程尝试关闭，将无法通过调用 `ngx.timer.at` 来创建一个新的非零延迟的计时器，并且此时 `ngx.timer.at` 将返回 `nil` 和 “process exiting” 错误。

这个 API 从 v0.9.3 版本开始，即使 Nginx 工作进程开始关闭的时候，仍然允许创建零延迟计时器。

当一个计时器到期时，计时器中用户定义回调的 Lua 代码将在一个与创建这个计时器的源请求完全隔离的 “轻线程” 中运行，所以，源请求生命周期内的对象，比如 `cosockets` 并不能与回调函数共享。

下面来看一个简单的例子：

```lua
location / {
 ...
 log_by_lua_block {
     local function push_data(premature, uri, args, status)
         -- push the data uri, args, and status to the remote
         -- via ngx.socket.tcp or ngx.socket.udp
         -- (one may want to buffer the data in Lua a bit to
         -- save I/O operations)
     end
     local ok, err = ngx.timer.at(0, push_data,
                                  ngx.var.uri, ngx.var.args, ngx.header.status)
     if not ok then
         ngx.log(ngx.ERR, "failed to create timer: ", err)
         return
     end
 }
}
```

还可以创建一个无限执行的计时器，例如，一个每 5 秒触发执行一次的计时器，在它的回调方法中递归的调用 `ngx.timer.at` ，这里给出这样的一个例子。

```lua
local delay = 5
local handler
handler = function (premature)
 -- do some routine job in Lua just like a cron job
 if premature then
     return
 end
 local ok, err = ngx.timer.at(delay, handler)
 if not ok then
     ngx.log(ngx.ERR, "failed to create the timer: ", err)
     return
 end
end

local ok, err = ngx.timer.at(delay, handler)
if not ok then
 ngx.log(ngx.ERR, "failed to create the timer: ", err)
 return
end
```

因为定时器的回调函数都是运行在后端，而且他们的运行时间不会叠加到客户端请求的相应时间中，它们可能会因为 Lua 语法错误，或者过多的客户端请求而很容易在服务端造成累积，或者耗尽系统资源。为了防止出现像 Nginx 服务器宕机这种极端结果，在一个 Nginx 工作进程中提供了对 “等待中的计时器” 和 “运行中的计时器” 这两种计时器的数量限制。这里 “等待中的计时器” 是指还没有过期的计时器，而 “运行中的计时器” 是指那些用户回调方法当前正在运行的计时器。

一个 Nginx 进程中所允许的 “等待中的计时器” 允许的最大数量由 `lua_max_pending_timers` 指令控制。而允许的 “运行中的计时器” 允许的最大数量由 `lua_max_running_timers` 指令控制。

目前的实现，每个 “运行中的计时器” 都会从 nginx.conf 配置中 `worker_connections` 指令配置的全局连接列表中占用一个 （虚） 连接记录，所以必须确保 `worker_connections` 指令设置了一个足够大的值能同时包含真正的连接数和计时器回调函数运行所需要的虚连接数（这个连接数是有 `lua_max_running_timers` 指令设限的）。

许多 Nginx 的 Lua API 能在计时器回调函数的上下文中使用，比如操作流和数据包的 cosockets API（`ngx.socket.tcp` 和 `ngx.socket.udp`），共享内存字典（`ngx.shared.DICT`），用户协程函数（`coroutine.*`），用户“轻线程”（`ngx.thread.*`），`ngx.exit`，`ngx.now/ngx.time`，`ngx.md5/ngx.sha1_bin`等都是可用的，但是相关子请求的 API （诸如`ngx.location.capture`），`ngx.req.* API`，下游输出 API （诸如 `ngx.say`，`ngx.print` 和 `ngx.flush`）都是明确在此上下文中不支持的。

你可以给计时器的回调函数传递大部分的标准 Lua 值类型（nils、布尔、数字、字符串、表、闭包、文件句柄等），要么显示的使用用户参数或者隐式的使用回调函数闭包的上游值。然而有一些例外诸如：你不能传递任何由 `coroutine.create` 和 `ngx.thread.spawn` 返回的线程对象，或者任何由 `ngx.socket.tcp`、`ngx.socket.udp` 和 `ngx.req.socket` 返回的 cosocket 对象，因为这些对象的生命周期是与创建他们的请求上下文绑定的，而计时器的回调函数（设计时）是与创建他们的请求上下文分离的，并且运行在它自己的（虚）请求上下文中。如果你试图跨越创建这些线程和 cosocket 的请求上下文边界来共享这些线程和 cosocket 对象，将会报错，对线程将报错 `no co ctx found`，对 cosocket 将报错 `bad request`，然而在计时器回调函数内部来创建这些对象则是没问题的。

这个 API 在 v0.8.0 版本第一次释出。


# ngx.timer.running_count
*语法：* `count = ngx.timer.running_count()`
*上下文：* `init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.*, balancer_by_lua*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*, ssl_session_store_by_lua*`

*返回当前正在运行的计时器数量。*
这个指令在 v0.9.20 版本第一次释出。


# ngx.timer.pending_count
*语法：* `count = ngx.timer.pending_count()`
*上下文：* `init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.*, balancer_by_lua*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*, ssl_session_store_by_lua*`

*返回当前正在等待的计时器数量。*
这个指令在 v0.9.20 版本第一次释出。