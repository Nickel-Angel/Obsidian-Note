SELinux 的基本原则是如果未写明允许，则禁止访问。
# 规则声明
首先 `.te` 文件的最基本的规则声明语法为：
``` te
rule_name source_type target_type:class perm_set;
```
其中，
`rule_name` 规则名：分别有 `allow, dontaudit, neverallow` 等；
`source_type` 源类型：发起操作的主体类型，通常是进程所属的域；
`target_type` 目标类型：被访问的客体类型，通常是文件，设备，socket 的类型；
`class` 类别：即被访问对象的类型，例如文件，设备，socket；
`perm_set` 权限集合：限制的各种权限。
几个示例：
```te
allow domain proc_cpuinfo:file r_file_perms;
dontaudit domain postinstall_mnt_dir:dir audit_access;
```
## rule_name
而具体来讲 `rule_name` 有几个比较常见的：
* `allow`：允许进程执行某个操作；
* `auditallow`：在权限检查成功时，仍然记录这个操作；
* `dontaudit`：在权限检查失败时，不记录此操作；
* `neverallow`：显式地写出某个动作不被允许，在编译层面上防止其他 allow 操作覆盖此规则。
## source_type & target_type
这两者各自代表了主体类型和客体类型，也即它们分别的域（或者叫做类型，我们一般把进程对应的 `type` 称为域，把文件对应的 `type` 称为类型）。而域可以通过 `type` 关键字来声明：
```te
type shell;
```
如上，我们就声明了一个名为 `shell` 的域。
由于经常会遇到不同的域需要相同的权限，为了方便，我们引入属性的概念。属性也可以作为规则声明语法的 `source_type` 或 `target_type`，从而对属性添加规则限制。属性可以通过 `attribute` 关键字来声明：
```te
attribute dev_type;
```
一个域可以和一个或多个属性相关联，被关联的域会吸收其所关联属性的所有权限。有两种关联方法，一种是在声明域时就将其和属性关联，一种是通过 `typeattribute` 关键字。
```te
type shell domain;

type httpd_user_content_t;
typeattribute httpd_user_content_t file_type, httpcontent;
```

而这些域是通过如下四个文件中的配置和具体的文件和进程关联起来的：
`mac_permissions.xml`：根据 apk 文件的签名，分配对应的 `seinfo` 标签；
`seapp_contexts`：根据应用的 `user`，`seinfo`，`name` 等信息，匹配对应的 `domain`（进程域）和 `type` 文件类型；
`file_contexts`：使用正则表达式指定某些文件，分配对应的文件类型；
`property_contexts`：为系统属性分配对应的域。
我们可以通过 `ls -Z` 查看某些文件的对应的安全上下文，其中就包含该文件的文件类型。安全上下文的格式如下：
```te
user:role:type:sensitivity[:category]
```
`user`：SELinux 的用户身份，常见的有 `u`（普通用户）和 `system_u`（系统用户）；
`role`：定义了可以担任的职责，常见的有 `r`（普通角色，如应用进程），`object_r`（客体角色，如文件），`system_r`（系统角色，如系统进程）；
`type`：定义了对象的 `type`；
`sensitivity`：敏感度，限制数据的可见性。
有了安全上下文的概念，我们也有一些关键字可以直接定义系统内一些对象的安全上下文。
比如虚拟文件系统的安全上下文，要使用 `genfscon` 来配置，其语法为：
```te
genfscon fs_type path_prefix [-file_type] context;
```
这里的 `context` 即安全上下文，会在下文介绍。
示例：
```
genfscon proc /mtk_demo/demo_file u:object_r:demo_context:s0;
```
网络对象也可以定义对应的安全上下文：
```te
portcon tcp 80 system_u:object_r:http_port_t;
netifcon eth0 system_u:object_r:netif_eth0_t system_u:object_r:netmsg_eth0_t;
nodecon 192.168.0.1 255.255.255.0 system_u:object_r:node_any_t;
```
## class
客体的具体类别，可以使用 `class` 关键字声明：
```te
class fd;
```
常见的 `class` 有这几种：
`file`：普通文件；
`dir`：目录；
`fd`：文件描述符；
`lnk_file`：链接文件；
`chr_file`：字符设备文件；
`binder`，`zygote`：Android 平台特有。
# te 中的正则表达式与集合
首先 te 中支持正则表达式，并且可以使用大括号来标识一个集合，对于集合支持 `* - ~` 三种通配符：
```te
# 排除 domain 中的 coredomain
allow { domain -coredomain } vendor_file_type:dir r_dir_perms;
# 不允许所有进程将文件挂载到 /system 文件和目录之上
neverallow * exec_type:dir_file_class_set mounton;
# 不允许所有进程加载除了集合以外系统文件
neverallow * ~{ system_file_type vendor_file_type rootfs }:system module_load;
```
# 类型转换规则
我们在权限管理过程中会遇到这样的问题：
由于进程在创建子进程的时候，子进程的权限默认与进程相同。而显然我们不希望这样，否则由于 linux 中所有进程都是 init 进程创建的，这样会导致所有进程都是相同的权限。
而同样由 init 进程生成的文件默认也是拥有较高的读写权限，这导致一些低权限的进程无法对文件进行读写。
我们通过类型转换来解决这些问题。
我们可以通过 `type_transition` 命令对域进行转换，使得其可以转到权限更高的域。语法为：
```te
type_transition source_type target_type:class default_type
```
## 主体的域类型转换
`type_transition` 这条命令只说明了转换的过程，没有授予相关权限，所以在实际中完整的使用应该是这样的：（这里假设我们要让 `init_t` 执行 `apache_exec_t` 文件的时候，新的进程切换到 `apache_t`）
```te
# 允许 init_t 执行 apache_exec_t 类型的文件
allow init_t apache_exec_t:file execute;
# 允许 init_t 的进程切换到 apache_t
allow init_t apache_t:process transition;
# 允许 apache_t 作为 apache_exec_t 的入口点
allow apache_t apache_exec_t:file entrypoint;
type_transition init_t apache_exec_t:process apache_t;
```
## 客体的转换
```te
type_transition passwd_t tmp_t:file passwd_tmp_t;
```
这里的意思是 `passwd_t` 在 `tmp_t` 目录下创建文件的时候，将文件的类型转换为 `passwd_tmp_t`。
这也需要两个前提条件：`source_type` 必须有创建文件并在该目录下添加文件的权限。
# 宏
我们可以通过宏将一些复杂的过程进行压缩，比如上文中主体的域类型转换前要做的操作：
```
define(`domain_auto_trans', `
	domain_trans($1,$2,$3)
	type_transition $1 $2:process $3
')

define(`domain_trans', `
	allow $1 $2:file { getattr open read execute }
	allow $1 $3:process transition
	allow $3 $2:file { entrypoint read execute }
	allow $3 $1:process sigchld;
	dontaudit $1 $3:process noatsecure;
	allow $1 $3:process { siginh rlimitinh };
')
```
以上是系统内部提供的主体的域类型转换的宏，同样也提供了 `file_type_auto_trans` 这样的客体转换的宏。
并且集合也可以定义成一个宏代替，当我们要使用宏定义的集合的时候，直接写出宏的名称即可。