# nft-forward.sh 变更历史 (CHANGELOG)

## v40
- `cmd_check` 在 `rules.conf` 为空时不再提前返回，始终执行第 7 节 `nft -c -f` 内核预演检查。
- `cmd_add` / `cmd_del` / `cmd_edit` / `current_rule_for_port` 支持匹配手写 `rules.conf` 中带前导零的端口（如 `08080`、`01000-01010`）。

## v39
- 将历史变更日志从脚本头部剥离至独立的 `CHANGELOG.md`，精简脚本本体。
- `existing_dnat_conflicts` 与 `cmd_check` 优先采用 `nft -j list ruleset`（JSON 结构化解析，需 `python3`），无 `python3` 或旧版 `nft` 不支持 `-j` 时自动回退到 `nft -n list ruleset | awk`。
- `cmd_check` 新增第 7 节：使用 `nft -c -f` 对生成的规则做只读内核语法/模块预演，提前发现容器缺 `nft_fib`/`nft_ct` 模块或语法不兼容问题。

## 历史版本记录 (v3 ~ v38)

```text
# nft-forward.sh v38 (最终版) — 多 VPS 通用 nftables 端口转发管理脚本 (一键安装 + 交互菜单 + 快捷命令)
# v38 修复 (基于 v37 二次复核):
#  1. 异段 1:1 端口转发改用逐端口规则 (兼容所有旧版 nftables / 内核, 消除大范围 map 拼接卡顿),
#     并限制异段转发最多 256 个端口 (同段转发仍为单条 `dnat to $tip`, 不限数量)。
#  2. cmd_update 的 curl 增加 `--connect-timeout 15 --max-time 60`, 避免网络异常时无限挂起。
#  3. cmd_edit 调整校验顺序: 先校验目标 IP/域名、目标端口与端口段长度, 再做新端口冲突检查与 22 二次确认。
#  4. existing_dnat_conflicts 遇到 `dport @命名集合` 时保守提示, 避免漏报。
#  5. valid_port 去除前导零后先判位数 <= 5, 消除超长数字触发 bash 整数溢出告警。
# v37 修复 (基于 v36 全面审计):
#  1. 跨端口段转发 (如 10000-10010 -> 20000-20010) 改为真正的 1:1 映射:
#     同段转发生成 `dnat to $tip` (保留原目的端口), 异段转发生成 `dnat to $tip : $proto dport map { ... }`
#     (原来 `dnat to IP:20000-20010` 在 netfilter 里是端口池随机/哈希分配, 不是 1:1)。
#  2. resolve_target 优先用 `getent ahostsv4` 解析 IPv4 (原来 `getent hosts` 对双栈域名
#     只返回 IPv6、不返回 A 记录, 精简系统没装 dig/nslookup 时会解析失败)。
#  3. cmd_edit / menu_edit: 先校验并归一化新本机端口再比对 (修 `08080` 自己撞自己);
#     端口冲突预查改用 ports_overlap 覆盖端口段重叠; 同端口 tcp/udp 指向不同目标时,
#     单协议修改不再误删另一协议规则。
#  4. existing_dnat_conflicts / check: `nft -n list ruleset` 避免服务名漏检;
#     匹配 `dnat (ip )?to` (覆盖 inet 表) 和 `th dport`; 只解析 dport 后的端口/集合,
#     避免把规则里 counter packets 8080 等数字误判为端口冲突。
#  5. rules.conf 全部 grep/sed 改用 `[[:space:]]+` 支持 Tab/多空格分隔; route_localnet
#     检查改用 awk 忽略注释行并支持 Tab。
#  6. apply_rules 补上 need_root; 清理重复 set -euo pipefail、distro_hint pipe 容错、
#     ss 冲突输出 || true、菜单规则计数去掉非法 both、update 版本号正则收紧、菜单 clear 不误报"已取消"。
# v36 (原版头): — 多 VPS 通用 nftables 端口转发管理脚本 (一键安装 + 交互菜单 + 快捷命令)
# v36 改进 (相对 v35, Kevin 问: 添加完规则自动生效, 为什么菜单还有第 5 项重新应用):
#  只改说明, 不改行为: 菜单第 5 项改成"重新应用规则 (手动改过配置文件 /
#  实时规则丢了才用)", 帮助里 apply 的注释也同步改清。添加/修改/删除本来
#  就自动保存并应用, 第 5 项只是修复按钮——手动改了配置文件、或者实时
#  规则被别的工具清掉后, 用它从配置文件重建; 重启后 systemd 自动恢复,
#  一般也不用按。
# v35 改进 (相对 v34, Kevin 要求: 菜单里加更新脚本选项, 第 9 项, 卸载和退出往后放):
#  新增 update: 菜单第 9 项"更新脚本 (从 GitHub 拉最新版)", 卸载移到第 10 项,
#  0 退出不变, 输入提示改成 [0-10]; 命令行 `nft-forward update` 也能用。
#  先下载到临时文件、校验 (bash -n + 版本号) 再替换, 任何一步失败都不动
#  现在用的版本。比对版本号: 一样就说已经是最新版; GitHub 上的更旧就提醒
#  先去 GitHub 覆盖上传; 更新才替换文件、跑一次 install (规则文件不动),
#  然后 exec 新版重开菜单 (终端里跑就进菜单, 非终端就只打印新版本号退出)。
#  失败用 return 1 不用 die, 菜单里调用时菜单不能被杀掉。下载地址和一键
#  安装器一样, 默认 GitHub raw, 可用 NFT_FORWARD_URL 环境变量改。
# v34 改进 (相对 v33, Kevin 要求: 改本机端口被拒绝后直接问新端口, 不用回菜单重来):
#  menu_edit 里新本机端口加预查循环: 本机端口不变时不查; 换端口时先查新端口
#  (已有别的规则 / 被监听/DNAT/Docker 占着 / 非法), 被拒绝就直接再问一次新端口,
#  回车取消。预查和 cmd_edit 里的检查是同一套, 改完仍会再查一遍才写文件。
#  22 不在这里预查, 由 cmd_edit 里的 22 确认处理 (取消后回菜单)。
#  目标/目标端口/解析/加载失败改端口也救不了, 报错后回菜单。
# v33 改进 (相对 v32, Kevin 反馈: 修改规则时只能改目标 IP/域名和目标端口, 不能改本机端口):
#  edit 加新本机端口: 菜单第 6 项先问"新本机端口 (回车 = 当前)", 再问目标/目标端口/
#  协议; 命令行 `nft-forward edit <本机端口> <目标IP或域名> [目标端口] [tcp|udp|both]
#  [新本机端口]`。换本机端口时新端口像 add 一样过检查: 已有别的规则占着就拒绝,
#  被监听/DNAT/Docker 占着就硬拒绝 (v32), 换成 22 要走 22 的确认 (跳过 ss 检查,
#  和 add 的 22 分支同一套)。本机端口不变时不查新端口 (改目标不该被后来占端口的
#  代理拦住)。仍是一次改完文件、一次 apply, 失败整条恢复。
# v32 改进 (相对 v31, Kevin 要求: 添加时端口被占用就硬拒绝, 不要问"确认还要继续"):
#  check_port_conflict 尾部: 任何一项冲突 (ss 监听 / 其他表 DNAT / Docker 发布)
#  都直接 die, 不再问"确认还要继续转发吗"。之前交互终端里输 y 可以强行继续,
#  现在占了只能换端口。只影响非 22 端口 (22 走单独分支, ss 监听本来就跳过)。
#  菜单添加时同样生效 (menu_add 调 cmd_add, die 后显示"添加失败")。
# v31 改进 (相对 v30, Kevin 要求: 菜单第 5 项后面加"修改转发规则", 改完自动保存并应用;
# 卸载移到退出前面):
#  新增 edit: 菜单第 6 项"修改转发规则" (命令行 `nft-forward edit <本机端口>
#  <目标IP或域名> [目标端口] [tcp|udp|both]` 也能用)。菜单里先显示当前值, 目标/
#  目标端口/协议三项回车就是保持不变; 命令行里目标端口/协议不填也保持当前值。
#  目标支持域名 (复用 v27 的 resolve_target, 添加时解析一次存 IP)。一次改完文件、
#  一次 apply (apply_or_restore, 失败整条恢复到修改前), 不是先 del 再 add——
#  del 完 add 失败会丢规则。菜单重排: 6 修改, 7 清空, 8 检查, 9 卸载 (紧贴退出
#  前面), 0 退出; 输入提示改成 [0-9]。
# v30 改进 (相对 v29, Kevin 问: 下载完没有任何提示, 成功了能不能显示版本号):
#  新增 version 子命令: `nft-forward version` 或 `bash nft-forward.sh version`
#  直接打印 "nft-forward v30"。不需要 root、不碰 nft, 下完文件就能先确认版本
#  再装; 多余参数同样拒绝 (和其他无参命令一致)。下载命令改成 curl 成功后
#  自动跑 version, 成功就有版本提示, 失败 curl 自己报错、version 不跑。
# v29 改进 (相对 v28, Kevin 又问: 没有 apt-get 时能不能加个安装 apt-get 的命令):
#  还是装不了——安装 apt-get 的命令就是 apt-get 自己 (apt-get install apt),
#  它不在, 命令就没法跑; 下面没有更底层的安装器了。硬从网上抓 .deb 用 dpkg
#  装可以写, 但版本必须跟系统发行版严丝合缝, 依赖缺一个就会把包数据库搞成
#  半安装状态, 不值得为几乎不存在的情况冒这个险。所以 v29 改的是报错:
#  check_nft 在 7 个包管理器全没有时, 按 /etc/os-release 认出系统, 有 dpkg
#  的 Debian/Ubuntu 会直接给 apt 官方池地址 (按发行版和架构拼好), 让用户
#  手动下 .deb 装; 别的系统或读不到发行版时也给一句能照着做的办法。
# v28 改进 (相对 v27, Kevin 问: 没装 nftables 时会不会自动装; 能不能 apt-get
# 也检测、没有就默认装 apt-get):
#  apt-get 本身装不了——它就是装软件的工具, 没包管理器就没法装包管理器。
# check_nft 的检测从 apt-get/dnf/yum 扩到 apt-get/apt/dnf/yum/apk/pacman/
# zypper 七个, 哪个在就用哪个 (覆盖 Alpine 的 apk、Arch 的 pacman、openSUSE
# 的 zypper, 以及只有 apt 没有 apt-get 的精简系统)。7 个全没有才报错。
# v27 新功能 (相对 v26, Kevin 在真机甲骨文 VPS 上实测: 菜单和 IP 目标都正常,
# 填域名 jpeastern.lindays.xyz 时报 "目标 IP 不合法"):
#  目标支持填域名。cmd_add 里新增 resolve_target: 已经是 IP 就原样用, 是域名
#  就 DNS 解析一次 (getent 优先, 依次 fallback nslookup/dig/host/python3,
#  取第一个 IPv4), 存解析到的 IP。只在添加时解析, 不在每次 apply 时解析——
#  开机时 systemd 跑 apply, 那时 DNS 可能还没好, 解析失败会让整批规则加载
#  失败。域名 IP 变了要 del 了重加 (那时再解析一次)。rules.conf 里仍只存 IP,
#  手写域名进规则文件仍会被 validate 拒绝。IPv6 地址和解析不到的域名都拒绝。
# v26 修复 (相对 v25, Kevin 传来另一份 v25, 我逐条实测它的 5 处改动, 发现其中 1 处引入严重回归):
# v26 修复 (相对 v25, 我逐条实测 v25 的 5 处改动, 发现其中 1 处引入严重回归):
#  v25 的第 4 项 (Docker 查询失败改为拒绝添加) 本意是对的, 但实现有 bug:
#  docker_published_conflicts 尾巴是 echo | grep -oE | while 循环, grep
#  找不到匹配时退出 1, 在 set -o pipefail 下整个函数返回 1, 调用方误以为
#  Docker 查询失败。实测 v25 在 docker 正常、端口没被发布时 add 也会被拒
#  ("无法确认 Docker 端口占用情况"), check 也误报风险 exit 1——有 docker
#  的机器上基本没法加规则。v26 在函数末尾显式 return 0 (真正查不了时上面
#  已 return 2)。其余 4 项实测都没问题, 原样保留。
# v25 修复 (基于 v24 复核, 另一 AI 的改动, 我逐条实测过):
# v24 修复 (相对 v23, Kevin 把脚本发给另一个 AI 审, 对方报了 3 个高风险问题;
# 我逐条在 v23 上实测复现, 2 个是真的、1 个在 v23 上已不存在):
#  1. [真] nft list ruleset 失败时, DNAT 冲突检查假装没冲突。existing_dnat_conflicts
#     在 nft 缺失或 list 失败时 return 0, add 会继续把规则加上, check 第 6 节
#     也报 "OK, 未发现碰撞"。v24 改成返回 2 表示"查不了": add 拒绝添加,
#     check 计一条风险 (exit 1)。
#  2. [真] add/del/clear 改完 rules.conf 后 nft 加载失败, 文件不回滚。
#     add 失败后文件里有规则、内核里没有; del/clear 失败后文件里没了、内核里
#     还在。v24 加 apply_or_restore: apply 在子 shell 里跑, 失败时把 rules.conf
#     从 .bak 恢复 (只在内核没动时恢复; nft 已加载但持久化失败时不恢复)。
#  3. [假, v23 已安全] install 在 apply 失败后继续。实测: nft -f 失败时 install
#     exit 1, "安装完成" 一次没出现, apply_rules 的 exit 1 直接结束整个进程。
#     未改。
# v23 修复 (相对 v22, 换角度排查: 菜单输入处理逐条核对, 发现 2 个问题):
#  1. 菜单里选 1 添加 (或选 2 删除) 后按 Ctrl-D, 脚本直接 exit 1、
#     没有任何提示。menu_add/menu_del 里 `read ... || return` 的裸
#     return 返回的是 read 的退出码 1, 在 set -e 下会杀掉整个菜单。
#     本来想写的是 "取消这次操作、回菜单", 所以必须 return 0。v23
#     四处都改成 return 0, Ctrl-D 后回菜单、菜单再读到 EOF 才干净退出。
#  2. 菜单顶上的 "当前规则: N 条" 把非法行也数进去了 (1 条真规则 +
#     garbage + 缺字段行, 显示 3 条)。v23 只数 4 字段且协议是
#     tcp/udp/both 的行, 和 list 的显示口径一致。
# v22 修复 (相对 v21, 换角度排查: check 各节逐条核对, 发现 3 个问题):
#  1. Docker 发布端口段时 check 第 6 节假放行。docker_published_conflicts
#     的正则只认单个端口, `0.0.0.0:10000-10010->10000-10010/tcp` 一行都
#     匹配不上, 段里的端口全漏报, add 的冲突检查 (同一函数) 也一样漏。
#     v22 改成端口段也认 (hp 校验同步放宽, ports_overlap 本来就支持段)。
#  2. 无 dport 的全端口 DNAT 漏报。existing_dnat_conflicts 要求行里有
#     "tcp dport ", `ip daddr X dnat to Y` 这种全端口 DNAT 直接跳过。
#     v22 这种行直接报出来 (不按端口比对)。同函数的表名匹配也由 index()
#     子串改成精确比对, 否则另一张表叫 x-nft-forward-nat 也会被当成自家表。
#     第 2 节显示的 awk 同样改精确匹配。
#  3. 第 4 节无 systemd 分支的关键词比 systemd 分支少了
#     flvx|miaomiao|wireguard|wgcf, 没 systemd 的机器上这几个代理查不到。
#     v22 对齐两支的关键词。
# v21 修复 (相对 v20, 换角度排查: status/list 在各种状态下的输出逐条核对, 发现 4 个问题):
#  1. nat 表没了但规则还在时, list 提示 "先跑 install 或 add" 是错的。
#     规则都在文件里, 该跑的是 apply (正好是 nftables restart 清表后的
#     恢复场景)。v21 按规则文件是否为空分开提示。
#  2. 服务文件存在但被 disable 时, status 说 "未安装" 是错的,
#     它明明装了只是没启用。v21 分开说。
#  3. filter 表没了、/etc/nftables.conf 不存在时, list/status 那几节
#     静默少一行, 像输出断了。v21 明说 "(filter 表未创建...)" 和
#     "(/etc/nftables.conf: 不存在...)。
#  4. 规则文件里手写多字段/少字段的行, list 的 printf 会把多余字段
#     当成新一行印出来。v21 字段不对就原样印出来并标注 "格式不对"。
# v20 修复 (相对 v19, 换角度排查: help 文本与实际行为逐条核对):
# v20 修复 (相对 v19, 换角度排查: help 文本与实际行为逐条核对, 发现 1 个问题):
#  1. clear / uninstall / 转发 22 的三次确认提示没有 [ ! -t 0 ] 守卫。
#     stdin 是永远不 EOF 的管道时 (某些自动化环境、ssh 非 tty 继承的
#     输入流), 裸 read 会一直挂起, 进程卡死、没有任何输出。ss 冲突提示
#     (check_port_conflicts) 早有这个守卫, 三处补上同样守卫, 非交互
#     直接报错退出。注意: 这也意味着 echo y | nft-forward clear 这类
#     管道喂 y 的写法不再生效, 破坏性操作必须在终端里跑。
# v19 修复 (相对 v18, 换角度排查: 卸载完整性):
# v19 修复 (相对 v18, 换角度排查: 卸载完整性):
#  1. 卸载后残留 /etc/nftables.conf.bak.nft-forward。卸载只删了 include
#     行、没删 install 时建的这个备份, 每次装卸循环都会留下一个。v19 在
#     换行恢复逻辑跑完后把它也删掉 (锁文件 /run/nft-forward.lock 不删,
#     锁文件惯例: 删了会让等锁的进程和新进程锁到不同 inode, /run 是
#     tmpfs、重启就清了, 留着无害)。
# v18 修复 (相对 v17, 换角度排查: 对抗输入与退出码):
# v18 修复 (相对 v17, 换角度排查: 对抗输入与退出码):
#  1. check 发现问题也返回 0。check 报了 "!! 碰撞 / !! 规则非法"、结论说
#     有风险, 但退出码永远是 0, 自动化里 check && install 会误以为没问题
#     就继续装。v18 在 bad>0 时 return 1, 干净时仍返回 0; 菜单第 8 项已
#     用 || true 接住, 不会因此闪退 (直接调 cmd_check 的话, set -e 会退
#     出菜单)。
#  2. 对抗输入已验证无注入: rules.conf 里写 $(...)/反引号/分号/多余字段/
#     格式串, apply 全部 exit 1 挡住, 没有任何命令被执行, rules.nft 里也
#     没有 flush 行; 恶意 add 参数同样被挡; list/check 只打印不执行。
#     (脚本里没有 eval, 规则字段进 nft 文件前必过 validate。)
# v17 修复 (相对 v16, 生产环境角度: 开机恢复):
# v17 修复 (相对 v16, 生产环境角度: 开机恢复):
#  1. 开机顺序竞态: 转发丢了要手动恢复。nft-forward.service 只有
#     After=network-online.target, 和 nftables.service 之间没有顺序。
#     装的时候 nftables 没启用 (所以没加 include), 之后用户手动启用
#     nftables.service 的话, 开机时 nftables 的 flush ruleset 可能在
#     nft-forward.service 的 apply 之后跑, 把刚建好的转发表清掉, 转发
#     就断了, 要手动 apply 才恢复。v17 加 After=nftables.service (只
#     有顺序, 没有 Wants/Requires, 不会拉起一个本来没启用的服务),
#     保证 nftables 先 flush, nft-forward 再 apply 重建。
# v16 修复 (相对 v15, 换角度排查: 环境差异与缺失命令):
# v16 修复 (相对 v15, 换角度排查: 环境差异与缺失命令):
#  1. flock 缺失时报错误导。acquire_lock 直接调 flock -w 10, 精简系统上
#     flock 不存在会返回 127, 被当成"拿不到锁", 报"另一个 nft-forward 正
#     在运行 (等了 10 秒还是拿不到锁)", 实际上根本没有别的实例在跑, 只是
#     没装 flock。v16 先查 command -v flock, 缺失就跳过加锁并提示一次只
#     跑一个命令 (ss/docker/timeout/nft 都有这类 guard, 唯独 flock 没有)。
# v15 修复 (相对 v14, 换角度排查: 并发与中断):
# v15 修复 (相对 v14, 换角度排查: 并发与中断):
#  1. 并行 add 大量丢失。flock -n 一碰上锁就失败, 10 个并行 add 只成 1 个、
#     剩下 9 个都要手动重跑 (规则本身没乱, 只是丢了活)。v15 改成最多等 10 秒
#     (flock -w 10), 前一个做完就轮到下一个; 10 秒还拿不到才报错。
#  2. 持久化文件不是原子替换。apply 最后一步用 cp -f 写 rules.nft, cp 会先
#     截断目标再写, 中途被杀会留下一个写了一半的 rules.nft, 下次开机 nftables
#     加载 include 时失败, 连 nftables 服务都起不来。v15 把临时文件建在规则
#     目录里 (和 rules.nft 同一文件系统), 最后一步用 mv 原子替换, 要么是旧
#     文件、要么是新文件, 不会有半截; 并在 apply 开头清掉上次被杀留下的临时文件。
# v14 修复 (相对 v13, 继续排查发现的剩余问题):
# v14 修复 (相对 v13, 继续排查发现的剩余问题):
#  1. nft 缺失且自动安装失败时没有明确报错。check_nft 里 apt-get/dnf/yum
#     失败后, 在 set -e 下脚本直接退出, 只剩包管理器自身的报错 (如 "E: 无
#     法获取锁"), 用户不知道是 nftables 没装上、install 没继续。v14 用
#     || true 接住安装命令的失败, 装完显式校验 nft 是否存在, 不存在就 die
#     一句明白话。
# v13 修复 (相对 v12, 再仔仔细细核对发现的剩余问题):
# v13 修复 (相对 v12, 再仔仔细细核对发现的剩余问题):
#  1. README 里的版本号还是旧的。标题写着 v7、菜单示例写着 v8, 脚本已是
#     v12, 都是升版时没跟着改的 (和 v11 修的脚本内版本号过期是同一类问题,
#     这次是在 README 里)。v13 把 README 标题、菜单示例、文件说明里的版本
#     号都改成 v13, 和脚本一致。脚本逻辑没动, 只改版本号和注释。
# v12 修复 (相对 v11, 再仔仔细细核对发现的剩余问题):
# v12 修复 (相对 v11, 再仔仔细细核对发现的剩余问题):
#  1. check 对非法规则假放行。rules.conf 里手写了 sctp、缺字段等非法规则时,
#     check 第 6 节不校验规则本身, 直接报 "OK, 未发现碰撞"、结论说"可以继续",
#     但 apply 根本加载不了这条规则。v12 在第 6 节逐条比对前先调 validate_rule
#     校验 (放子 shell 里跑, 失败不退出 check), 非法就标 "!! 规则非法" 并计入
#     bad, 结论不再说可以继续。
# v11 修复 (相对 v10, 再仔仔细细核对发现的剩余问题):
# v11 修复 (相对 v10, 再仔仔细细核对发现的剩余问题):
#  1. 用户能看到的字样里还写着 v7, 脚本已是 v10。报错信息 "(v7 只支持 IPv4
#     目标)"、list 的 "(v7: 只有 nftables 服务本来就启用时才加...)"、help 里
#     的 "v7 只支持 IPv4" 和 "v7 不主动 enable" 共 5 处, 都是 v7 时期写死的
#     版本号, 之后升版没跟着改。v11 把用户可见字样里的版本号全去掉 (只留
#     注释里的 v7/v8/v9 当历史记录), 以后再升版也不会又过期。
#  2. (新验证手段, 不是脚本 bug) 这次试了用 unshare 建 user+net namespace
#     跑真 nft: 不带 fib/ct 的 DNAT 和链结构能正常加载, 说明生成的 nft 文件
#     语法没错; 但 `fib daddr type local`、`ct status dnat`、`ct state` 这三
#     类规则在本容器里都报 "No such file or directory", 原因是本容器内核没
#     有 nft_fib / nft_ct 模块 (/lib/modules 不存在、modinfo 找不到)。这是测
#     试环境的限制, 不是脚本问题 (这些都是标准表达式, 正常 VPS 内核都有),
#     但也说明脚本核心安全规则 (fib+ct) 在这边至今仍没在真内核上跑通过,
#     仍需在真 VPS 上验证。
# v10 修复 (相对 v9, 仔仔细细逐行核对发现的剩余问题):
# v9 修复 (相对 v8, 再仔细检查发现的剩余问题):
#  1. status 输出里有没头没尾的 "enabled"/"disabled"。cmd_status 用
#     `systemctl is-enabled X 2>/dev/null || echo ...`, systemctl 在 stdout
#     上本来就会打印 enabled/disabled, 2>/dev/null 只挡 stderr, 所以
#     status 里会冒出光秃秃的一行 "disabled" 或 "enabled", 没标签、看不懂。
#     v9 改成 `is-enabled --quiet` (只返回退出码、不打印), 再按结果打印
#     带标签的中文状态, 启用/未启用都看得明白。
#  2. (顺带确认, 不是 bug) 手写 rules.conf 时 both / 大写 TCP / 多余字段
#     都会被 validate 拒掉 (both: 协议只能是 tcp/udp; 大写 TCP 同样拒;
#     多余字段会进到目标端口里、端口校验失败), Tab 分隔的字段正常通过;
#     目标写成本机自己的 IP 时是本地端口重定向 (18080 -> 本机:80), 不是
#     死循环, 不拦截, 只在 README 里注明。
# v10 修复 (相对 v9, 仔仔细细逐行核对发现的剩余问题):
#  1. [高危, 报错全丢] acquire_lock 里 `exec 9>"$LOCK_FILE" 2>/dev/null` 会把
#     整个进程的 stderr 永久指到 /dev/null (exec 不带命令时重定向作用于当前
#     shell)。add/del/clear/apply/install/uninstall 都先调 acquire_lock, 所以
#     之后所有 die 的报错都看不见, 只剩一个光秃秃的 exit 1。实测: add 99999、
#     add 999.1.1.1 都只返回 exit 1、没有任何报错信息; 直接测 `exec 9>锁 2>/dev/null`
#     后 `echo ... >&2` 也确实没输出。v10 去掉这行上的 2>/dev/null (打不开锁文件时
#     bash 自己的报错可见, || 兜底仍会走跳过加锁的分支)。uninstall 里的
#     `exec 9>&- 2>/dev/null` 同样会吞 stderr, 一并去掉。
#  2. [中] 规则文件里只有空格/Tab 的空行会被当成规则, apply 直接报"缺字段"失败;
#     check 第 6 节会打印 "检查:  -> :" 的空行还报 OK; list 会打印空白行。
#     v10 把 6 处读文件循环的空行判断改成"全空白也算空行", 和注释行一样跳过。
#  3. clear 在没安装过时 (RULES_DIR 不存在) 会因 `: >` 目录不存在而失败退出。
#     v10 先 mkdir -p, 和 cmd_add 保持一致。
#  4. 头部注释里 v8 标题行重复出现两次, 删掉一行 (纯注释, 不影响功能)。
#  5. backup_rules 在规则文件不存在时返回 1, 在 set -e 下调用它的命令会直接
#     退出且没有任何报错。实测: 没安装过时 clear 直接 exit 1、无输出 (trace
#     到 backup_rules 的 `[ -f ]` 就停了)。v10 改成 if 写法并显式 return 0。
# v8 修复 (相对 v7, 再仔细检查 + shellcheck 发现的剩余问题):
#  1. shellcheck 警告全部清掉 (之前 warning 级别有 1 个 warning + 2 个 error)。
#     - valid_ip 的 `local -a o=($ip)` 改成 `IFS='.' read -ra o` (SC2206)。
#     - check 里 `grep -E ":$pt[[:space:]]"` 改成 `":${pt}[[:space:]]"`,
#       消除 $pt[...] 被当成数组展开的歧义 (SC1087, 虽然实测没错, 但含糊)。
#     - 两处 `A && B || die` 改成显式 if (SC2015, 原写法意图是对的, 但
#       A 真 B 假时 C 也会跑, 换成 if 更不容易看错)。
#     - include 追加时 `tail | read` 和 `>>` 写同一个文件改成先读到变量
#       再追加 (SC2094)。
#  2. docker ps 卡住会拖住 add/check。docker daemon 僵住时 `docker ps`
#     可能一直不返回, add/check 就会一直等。v8 给 docker ps 加 5 秒超时
#     (有 timeout 命令时用, 没有就按原来跑), 超时按"查不到"处理, 不阻塞。
#  3. 原文件末尾没换行时, 卸载恢复不成原样。install 会在原内容后补一个换行
#     再追加 include, 卸载时只删 include 两行, 补的那个换行留下来了, 文件
#     末尾就多了一个换行。v8 在卸载后拿备份文件对照, 备份末尾没换行、当前
#     文件末尾有换行时, 去掉末尾一个换行, 恢复成字节级原样 (mock 实测: 有/
#     无末尾换行两种原文件, 卸载后 diff 都干净)。同时 install 每次都刷新
#     备份 (原来只在备份不存在时才建, 重装后备份会是旧文件的, 对照就错了)。
# v7 修复 (相对 v6, 再仔细检查发现的剩余问题):
#  1. 重复/重叠的转发规则会被悄悄接受。手写 rules.conf 时,
#     `tcp 18080 -> 1.2.3.4:80` 和 `tcp 18080 -> 9.9.9.9:90` 会同时
#     生成两条 DNAT, 第一生效、第二条是死规则, 用户以为两条都通;
#     `tcp 18080` 和 `tcp 18080-18090` 重叠也一样。v7 在 validate 时
#     按协议查本机端口重叠, 有重叠直接报错, 不生成歧义规则。
#     (add 命令本身有去重, 所以这只防手写文件, 但手写文件是常见用法。)
#  2. 目标 IP 0.0.0.0 / 255.255.255.255 会被接受。DNAT 到 0.0.0.0
#     是无效地址, 到 255.255.255.255 是广播地址, 真机 nft 加载时
#     要么报错、要么建出一条不通的规则, mock 里却显示"已添加"。
#     v7 在 validate 和 add 里都拒掉这两个地址 (以及 0.x.x.x)。
#  3. 其他表用集合写法的 DNAT 查不到。`tcp dport { 18081, 18082 } dnat`
#     或 `udp dport { 19000-19010 } dnat` 时, 旧的检查只认 `dport 数字`
#     这种写法, 集合里的端口一个都查不到, add 就直接放行、抢走别人的
#     流量。v7 把冲突检查改成在 awk 里直接解析集合 (单端口、范围、
#     逗号分隔的多个端口/范围都认), 有重叠就报警。
# v6 修复 (相对 v5, 再检查发现的剩余问题):
#  1. del 不存在的规则也报"已删除"。`del 19999` 和 `del 18080 udp` (只有 tcp
#     时) 都会 exit 0 并绿字"已删除", 实际上文件里什么都没动, 还白跑一次
#     apply。v6 先查规则在不在, 不在就直接报错"没有这条规则", 不备份、
#     不 apply、不谎报成功。
#  2. 多余参数被静默忽略。`add 8080 1.2.3.4 80 tcp extra` 会成功并丢掉
#     extra; 更危险的是 `clear foo` 也会真的清空全部规则。v6 在 add/del
#     里拒掉多余参数, 在分发处对 list/status/check/clear/apply/uninstall
#     等无参命令也拒掉多余参数。
#  3. 前导空格的规则删不掉。rules.conf 里 `  tcp 18080 ...` (行首有空格)
#     时, sed 的 `^tcp` 匹配不上, del 报"已删除"但规则还在, add 同端口
#     时去重也删不掉旧行、会写出重复规则。v6 的 sed 改成允许行首空白,
#     且 add/del 的存在性检查也用同样宽松的匹配。
# v5 修复 (相对 v4, 继续检查发现的剩余问题):
#  1. [高危, Docker 检查完全失效] docker_published_conflicts 里的
#     `tr -d ':->'` 把 '-' 当成了字符范围 ':' 到 '>' (ASCII 58-62), 删不掉 '-',
#     导致主机端口变成 "8080-" 这种带横杠的字符串, 后面的 ^[0-9]+$ 校验永远
#     失败、直接 continue, Docker 冲突检查从 v2 起就从没报过任何一次。
#     v5 改用 grep -oE ':[0-9]+->[0-9]+/(tcp|udp)' 直接取主机端口和协议,
#     顺便修了协议误报: 之前不分 tcp/udp, Docker 只发布了 tcp 8080 时,
#     加 udp 8080 也会误报; 现在只报同协议的冲突。
#  2. [中危] rules.conf 带 Windows 换行 (\r\n) 时 apply 会失败。
#     read 出来的 tport 会变成 "80\r", valid_port 的 ^[0-9]+$ 通不过,
#     报错还看不出 \r, 很难排查。v5 在 validate/build/check 的每个读文件
#     循环里都先去掉行尾 \r, 并在 validate 失败时把 \r 显示为 <CR>。
#  3. [中危] 端口段前导零算错/崩溃。port_len 用 $((020-010+1)) 会把 010 当
#     八进制, 010-020 算成 9 而不是 11; 08-09 直接报 "value too great for
#     base" 崩溃。v5 在 port_len / ports_overlap 里都用 10# 强制十进制。
#  4. [中危] add 端口段时静默丢弃用户填的目标端口。`add 10000-10010 1.2.3.4 80`
#     会被悄悄改成 10000-10010 (1:1), 用户填的 80 不声不响就没了。
#     v5 删掉这段静默覆盖, 只留"不填目标端口时默认跟本机端口一样"这一条;
#     显式填了单个 80 时, 长度校验会直接报错, 告诉用户段只能转同样数量的段。
#  5. uninstall 不加锁, 可能和正在跑的 apply/add 并发删表/删规则文件。
#     v5 开头就 acquire_lock, 删本体前先关掉锁 fd (否则后台删脚本的进程
#     会继承锁 fd, 锁住 1 秒)。
#  6. 转发 22 端口时跳过全部冲突检查 (包括其他表 DNAT 和 Docker)。
#     v5 在 22 确认之后, 仍查其他表 DNAT 和 Docker 冲突, 有冲突直接拒绝;
#     只跳过 ss 监听检查 (sshd 本来就占着 22, 那是预期内的)。
# v4 修复 (相对 v3, 最终核对发现的剩余问题):
#  1. install 不再用 systemctl enable --now。v3 的 install 在持有 flock 锁的情况下
#     --now 启动 nft-forward.service, 那个服务进程也要拿同一把锁, 会直接失败。
#     v4 只 enable (不开 --now), 立即生效靠 install 里直接调的 apply_rules, 不靠服务启动。
#  2. include 增加 flush 检查。v3 只查 nftables.service 是否启用就追加 include,
#     但如果 /etc/nftables.conf 里没有 "flush ruleset", 重启加载时表已存在会
#     因 "table already exists" 导致整个 nftables 服务加载失败。
#     v4 要求文件里必须有 flush ruleset 才加 include, 否则跳过并提示只靠本服务恢复。
#  3. check 命令的 DNAT 检查不再误报。v3 里 nft list 失败时, grep 空结果也会显示
#     "(无其他 DNAT)", 会让人误以为已检查过。v4 先判 nft list 是否成功, 失败就明确说
#     "无法检查, 请用 root 且有 NET_ADMIN 权限再跑", 不再假装无冲突。
#  4. filter 放行规则的匹配顺序统一为 ip daddr 在前, 与 masquerade 规则一致,
#     避免个别 nft 版本对匹配顺序的解析差异。
#  5. [高危, 假碰撞] DNAT 冲突检查只认真正的 "dnat to" 行, 且排除本表的两个表。
#     v3 的检查用 grep "dnat" 会把 "ct status dnat masquerade/accept" 也当成 DNAT,
#     且只排除 nft-forward-nat 表、不排除 nft-forward-filter 表, 导致本地端口
#     等于目标端口的规则 (如 51820->51820) 在 check 里报假碰撞, 甚至会拦住
#     同端口覆盖重加。v4 用 awk 按表名排除 nat+filter 两个表, 且只认 "dnat to"。
#     check 第 2 节同样修: 之前 grep -v nft-forward 排除不了本表 DNAT (DNAT 行
#     本身不含表名), 会把本表规则也列为"其他表 DNAT"。
# v3 的修复全部保留: DNAT 加 fib daddr type local、MASQUERADE 加 ct status dnat、
# 不主动 enable nftables.service、route_localnet 按需开启、端口冲突查 DNAT+Docker、
# check 命令、forward policy accept、原子加载等。
# 仍不支持 IPv6 转发 (只支持 IPv4 目标)。
#
# 一键安装:  bash nft-forward.sh install   (或直接 bash nft-forward.sh, 无参数自动进菜单)
# 呼出菜单:  nft-forward   或   zf   (安装后任意目录直接输)
# 命令行:    nft-forward add 8080 1.2.3.4 80 tcp
# 上线前检查: nft-forward check
# v25 修复 (基于 v24 复核):
#  1. del 参数未校验就拼入 grep/sed 正则, `del .*` 等输入可能误删多条规则。
#     现在先校验端口并规范化, 阻断正则注入/误删。
#  2. 脚本只支持 IPv4 转发, install 却无条件开启 IPv6 forwarding, 可能改变
#     VPS 的 IPv6 路由/RA 行为。移除无关 IPv6 sysctl 修改。
#  3. clear 在规则文件原先不存在时没有备份, apply 失败无法恢复原始状态。
#     现在先创建空规则文件再备份, 失败时可恢复为空文件。
#  4. Docker CLI 存在但 daemon 查询失败/超时时, 原来被当成"无冲突"。
#     改为明确报错并拒绝 add, check 也将其计为风险。
#  5. 拒绝 IPv4 八进制/歧义形式的前导零, 避免不同解析器对地址含义不一致。

#  1. [高危, 代理失效根因] DNAT 只对本机地址生效: 加 fib daddr type local。
#     v2 的 DNAT 是 "$proto dport X dnat to ..." 不限目的 IP, 会劫持所有经过本机
#     prerouting 且目的端口相同的流量, 包括:
#       - Docker 容器出站、Xray/mihomo 出站、WireGuard/WARP 出站要去外部服务器的流量
#       - 经本机转发的其他代理流量 (WireGuard 对端、TUN 流量、内网客户端经本机去外网)
#     只要目的端口撞上转发端口, 就会被改写到转发目标, 代理就断了。
#     修后只有目的地址是本机 IP (公网 IP / 私网 IP / 云厂商 NAT 后翻译成的私网 IP)
#     的包才会被 DNAT, 其他流量原样通过。这是 Docker / systemd 用的标准写法。
#  2. [高危] MASQUERADE 只对被 DNAT 过的连接生效: 加 ct status dnat。
#     v2 的 masquerade 是 "ip daddr 目标 tcp dport 端口 masquerade", 会改写本机代理
#     进程自己主动去连同一个目标 IP:端口时的源地址 (如 Xray 出站连落地机)。
#     修后只有 ct status dnat 的包才 masquerade, 本机出站流量不动。
#     forward 链的放行规则同样加 ct status dnat, 只放行被 DNAT 过的流。
#  3. [高危] 不再 enable nftables.service。v2 的 install 会 systemctl enable
#     nftables.service, 可能激活一台之前处于禁用状态的默认防火墙配置
#     (/etc/nftables.conf 里自带的 filter 表), 下次开机就把代理端口挡掉。
#     v3 不动 nftables.service 的启用状态, 只用自己的 nft-forward.service 开机恢复。
#     自己的服务也不再 After=nftables.service, 只 After=network-online.target。
#  4. include 只在 nftables.service 本来就已启用时才加。
#     v2 是只要 /etc/nftables.conf 存在就追加 include; v3 先查
#     systemctl is-enabled nftables.service, 本来没启用的就不改那个文件,
#     只靠自己的服务开机恢复, 避免把一个 dormant 的配置激活。
#  5. route_localnet 不再安装时全局开启。v2 的 install 就写
#     net.ipv4.conf.all.route_localnet=1, 会放宽 127.0.0.0/8 的路由。
#     v3 只在 apply 时、且确实有规则要转到 127.0.0.1 时才开, 没有这种规则就不开。
#  6. 端口冲突检查补上 ss 查不到的两类:
#     a) 其他表已有的 DNAT: 查 nft list ruleset 里非本表的 dnat 规则, 同端口/端口段
#        重叠就报警 (Docker / K8s / 其他面板的 DNAT 在 ss 里看不到监听端口)。
#     b) Docker 已发布端口: 查 docker ps 的 Ports 列, 已发布的主机端口同样报警。
#     之前只查 ss, Docker 容器端口和别的面板的 DNAT 是查不到的。
#  7. 新增 check / preflight 命令和菜单第 8 项: 上线前检查。
#     一次性列出本机所有监听端口、其他表的 DNAT、Docker 发布端口、代理相关服务
#     (xray/sing-box/mihomo/wg/docker/nginx 等)、本机 IP, 并逐条比对现有转发规则
#     有没有撞上, 没撞上才建议继续。
# v2 的修复全部保留: forward policy accept、原子加载、端口段长度校验、sysctl 容错、
# 加锁+备份、菜单终端判断、卸载延迟删本体、systemd 缺失容错。
# 仍不支持 IPv6 转发 (只支持 IPv4 目标)。
#
# 一键安装:  bash nft-forward.sh install   (或直接 bash nft-forward.sh, 无参数自动进菜单)
# 呼出菜单:  nft-forward   或   zf   (安装后任意目录直接输)
# 命令行:    nft-forward add 8080 1.2.3.4 80 tcp
# 上线前检查: nft-forward check
```
