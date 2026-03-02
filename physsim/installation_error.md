下面我按时间线把我们从一开始到现在遇到的主要 bug / error 以及对应修复策略完整汇总一遍（尽量做到你以后只看这一条就能回忆起来发生了什么、为什么会那样、怎么解决）。

⸻

A. 最早阶段：rpm -i *.src.rpm 的 “fujiik / ilc_ilc 不存在” 警告

现象

你在 NAF 上执行：

rpm -i pythia-6.428-kekcc2c.src.rpm

反复出现：
	•	warning: user fujiik does not exist - using root
	•	warning: group ilc_ilc does not exist - using root

根因（本质）

SRPM 里带了文件属主/属组元数据（KEK 环境里存在 fujiik, ilc_ilc），NAF 上不存在这两个名字，rpm 在解 SRPM 时就会发出 warning，并用 root 代替。

修复/处理

不需要修复，属于可忽略 warning。
这一步关键是 SRPM 解包成功并生成 .spec、源 tarball/patch 被放进 rpmbuild tree；warning 不影响后续 rpmbuild。

⸻

B. rpm --eval '%{_topdir}' 指向 home (AFS) vs DUST：你不想污染 home

现象

一开始 rpm --eval '%{_topdir}' 返回：

/afs/desy.de/user/z/zhangyuy/rpmbuild

而你想把 rpmbuild/、sw/ 等都放在 DUST（干净、不会忘、不会挤爆 home quota）。

根因

RPM 默认 topdir 在 ~/rpmbuild（你的 home/AFS）。

修复

你创建了 ~/.rpmmacros 并设置：

%_topdir /data/dust/user/zhangyuy/analysis/physsim/rpmbuild

从此 rpm --eval '%{_topdir}' 都回到 DUST：

TOPDIR=/data/dust/.../physsim/rpmbuild

这是非常关键的一步：它把所有 BUILD/BUILDROOT/RPMS/SRPMS/SOURCES 都锁定在 DUST，避免“我是谁我在哪儿”的 ADHD 灾难。

⸻

C. “home/ 不存在”导致 mv home/<uid>/sw/... 失败（pythia6 初始困惑）

现象

按照 KEK 指令解 RPM：

rpm2cpio ... | cpio -id
ls
home usr
ls home/
fujiik

所以 home/<your user id> 根本不存在。

根因

这些 RPM 的 %install / Prefix 是在 spec 里固定为 KEK 的路径（典型是 /home/ilc/fujiik/sw/...），因此 rpm payload 里自然落在 home/fujiik/...，而不是你的用户名目录。

修复（你现在已经掌握的通用解法）

不用硬写用户名；用通配符定位真实路径：

mv home/*/*/sw/pythia "$PHYSROOT/sw/"

	•	home/*/*/sw 会匹配 home/fujiik/ilc/sw 或 home/ilc/fujiik/sw 等多种结构
	•	从而绕开“KEK 用户名不匹配”问题

并且你也追问过：* 是 glob wildcard，匹配任意字符串；如果出现多个匹配会怎样——结论是：要么用 ls home/*/*/sw 先确认只有一个目标，要么改成 find ... -maxdepth ... -name pythia -type d 做唯一定位（这是更稳的版本）。

⸻

D. “ls -l “$PHYSROOT/sw/ … q exit””——你被 shell 卡住

现象

你在 shell 里输入了一个不完整的引号/路径：

ls -l "$PHYSROOT/sw/

shell 会进入 continuation 状态（等待你补全引号），你看到提示符像 >，你就开始输入 q exit 等，都没用。

根因

Bash 解析到未闭合的双引号会等待继续输入（不是进入某个程序，而是命令行语法没闭合）。

修复

直接按 Ctrl+C 终止当前未完成输入，回到正常提示符。
以后看到 > 这种 continuation prompt，第一反应就是：是不是少了引号/括号/反引号。

⸻

E. rpmbuild 的 ctrl+C / tail 的误解：你以为没结束其实早结束了

现象

你在 tail -f LOGs/root5 时觉得它“卡在最后一行很久”，后来发现其实 rpmbuild 已经 Done，只是 tail -f 没退出。

根因
	•	tail -f 只负责“跟随文件变化”；当构建结束日志不再变化，它就一直挂着，看起来像“停在最后状态”。
	•	你按 Ctrl+C 只是停止 tail，不是停止后台 rpmbuild（除非你 ctrl+c 在前台直接运行 rpmbuild 的窗口里）。

修复（经验规则）

判断 rpmbuild 是否结束，不靠 “tail 停住了没有”，靠：
	1.	看后台 job：jobs -l
	2.	或 ps -u $USER | grep rpmbuild
	3.	或更硬：ls $TOPDIR/RPMS/x86_64 | grep <pkg> 是否已经出现目标 rpm
只要 Wrote: ...RPMS/...rpm 出现，就是成功结束。

⸻

F. LCIO 构建失败：pushd /home/ilc/fujiik/sw/root/5.34.36: No such file or directory

现象

lcio build log 中：

rootdir=/home/ilc/fujiik/sw/root/5.34.36
pushd /home/ilc/fujiik/sw/root/5.34.36
No such file or directory
error: Bad exit status (%build)

根因

lcio.spec 里需要 root5 的路径（用于 ROOT dictionaries / root-config 等），但它仍然在引用 KEK 的固定路径，而你的 root5 安装在：

/data/dust/user/zhangyuy/analysis/physsim/sw/root/5.34.36

修复

你检查 spec 并确认：
	•	旧的 /home/ilc/fujiik/sw/root 并不存在或不该用
	•	spec 里已经有 %define __root_dir /data/dust/.../sw/root/5.34.36

最后你成功 rpmbuild 生成：

lcio213-02.13.01-...x86_64.rpm

（也就是说：核心修复策略就是“查 spec 里硬编码路径→改成你的 PHYSROOT/sw 前缀”）

⸻

G. openjade 失败：找不到 opensp 头文件（硬编码 opensp 路径）

现象

openjade build 时：

fatal error: /home/ilc/fujiik/sw/opensp/1.5.2/include/OpenSP/config.h: No such file or directory

根因

openjade.spec 里 __optdir 定义为 /home/ilc/fujiik/sw，因此 __ospdir 指向 KEK 的 opensp 安装；但你 opensp 实际装在：

$PHYSROOT/sw/opensp/1.5.2

修复

不是简单替换某一行 configure，而是改 宏定义链：
	•	修改 %define __optdir ... 或更直接：把 __ospdir 生成路径改成你的 $PHYSROOT/sw/opensp/...
	•	清理旧 BUILD/BUILDROOT
	•	重跑 rpmbuild

最终 openjade 成功产出 rpm，并且你也成功把 openjade 目录搬到 $PHYSROOT/sw/openjade/...。

⸻

H. ocamlfind4 最复杂的一组错误与修复（你们迄今最大坑）

这段是最值得总结的，因为它包含多个子问题，几乎每一个都踩过。

H1. 第一次 ocamlfind4 build：File not found: .../home/ilc/fujiik/sw/ocaml/4.14.2/bin/* 等

现象
rpmbuild 在 %files 检查时报：
	•	Directory not found / File not found：.../home/ilc/fujiik/sw/ocaml/4.14.2/...

根因
ocamlfind4 实际 make install DESTDIR=%{buildroot} 安装到了 buildroot 下的 /usr/… 路径（或部分路径），而 spec 的 %files 却写的是 KEK 的 %{prefix}（/home/ilc/fujiik/sw/ocaml/4.14.2）树。

修复策略（第一次尝试）
你往 %install 末尾追加了一段 “mirror usr tree into expected ocaml prefix” 的补丁块，把 buildroot 里的 /usr/... 拷贝到 buildroot 里的 /home/ilc/fujiik/sw/ocaml/4.14.2/...，以满足 %files。

但中途你又把部分路径写成了 .../data/dust/...，导致 %files 仍找不到 KEK prefix。

⸻

H2. ocamlfind4 configure 失败：cannot determine ocaml's standard library directory

根因
ocamlfind 的 configure 需要正确识别 ocamlc -where，但你构建环境里的 OCaml 工具链在一些地方仍携带 KEK 前缀（尤其 runtime/stdlib 相关）。

修复
在 spec 中，在 ./configure 前明确注入：
	•	export OCAMLLIB=.../lib/ocaml
并保证 PATH 指向你的 ocaml bin。

⸻

H3. ocamlfind4 configure/ocargs.log：bad interpreter: .../home/ilc/fujiik/.../ocamlrun

现象
ocargs.log 显示：

tools/extract_args/extract_args: /home/ilc/fujiik/sw/ocaml/4.14.2/bin/ocamlrun: bad interpreter

根因
你安装的 ocaml 可执行文件/脚本（或它调用的 runtime）内部仍然硬编码了 /home/ilc/fujiik/.../ocamlrun，在 NAF 上不存在这个路径。

修复
两层修复叠加：
	1.	你在自己的 ocaml 环境中把引用旧 ocamlrun 的地方替换为 DUST 的 ocamlrun（你做了 grep+sed）。
	2.	在 spec 里显式设置：
	•	OCAMLRUN=$PHYSROOT/sw/ocaml/4.14.2/bin/ocamlrun
	•	CAML_LD_LIBRARY_PATH=$OCAMLLIB/stublibs:...
让运行时总能找到正确 runtime 与 stublibs。

⸻

H4. ocamlfind4 build：链接失败（大量 undefined reference to unix_*）

现象
构建 ocamlfind 时：
	•	一大串 undefined reference to unix_*
	•	最终 Error while building custom runtime system

根因
它在用 ocamlc -custom 构建 custom runtime（把 unix runtime 链进去），但你的环境/链接参数组合导致链接不到对应符号（在某些系统上这类 custom runtime build 很容易坑）。

修复（关键转折）
你采用了“强制去掉 -custom”的方案，并最终成功：
	1.	写 wrapper：work/wrapbin/ocamlc
	•	内部调用真实 ocamlc
	•	过滤掉 -custom
	•	并显式 -use-runtime <dust>/ocamlrun
	2.	在 spec 里把 PATH 调整为 wrapbin 优先
	3.	同时你还尝试在源码 Makefile 里 sed 掉 -custom（这步有些地方没完全生效，但 wrapper 最终兜底成功）

⸻

H5. ocamlfind4 最后阶段：Installed (but unpackaged) file(s) found 与 %files 路径混乱

现象
	•	rpm 报大量 “Installed but unpackaged file(s) found: /data/dust/…”
	•	同时 %files 还在期望 /home/ilc/fujiik/sw/ocaml/...

根因
你让 ocamlfind 的 install 直接写进了 buildroot 里的 DUST 路径（例如 buildroot/data/dust/...），这会被 rpm 视为“包外文件”（因为 %files 没列它们），并且这种绝对路径在 rpm 里也不合适。

修复（最后收敛）
把策略统一成两条“原则”：
	1.	构建期间可以用 DUST ocaml 运行时（PATH/OCAMLLIB/OCAMLRUN），但打包内容必须落在 buildroot 的 KEK prefix（/home/ilc/fujiik/sw/ocaml/4.14.2/...），以匹配 %files。
	2.	%install 末尾的 “mirror” 必须拷到 buildroot 下的 /home/ilc/fujiik/...（而不是拷到 buildroot 下的 DUST 路径）。

最终 rpmbuild log 出现：
	•	Wrote: .../RPMS/x86_64/ocamlfind4-1.9.8-...rpm
并且没有再报 Installed (but unpackaged)，说明打包闭环完成 ✅

你最后贴的输出就是这个成功版本。

⸻

I. “rpm 文件很小是不是没编译完全？”（pythia6 1.6MB 的疑问）

现象

你看到：
	•	pythia-6.428...x86_64.rpm 只有 ~1.6MB

根因

pythia6 本体很旧、内容很少（主要是库+少量接口），而且 spec 往往只打包必要文件，不像 root5 那样巨大。

修复/结论

不是问题。判断完整性只看：
	•	Wrote: ...RPMS/...rpm 是否出现
	•	rpm -K --nosignature <rpm> 是否 digests OK
	•	解包后目录结构是否完整（你看到 $PHYSROOT/sw/pythia/v6.428）

⸻

一句话总括（你应该记住的“通用解题法”）

你在 NAF 里装 KEK 的这些 RPM 包时，真正的核心困难只有一种：spec 里硬编码的 KEK 前缀路径。
我们处理它的套路也只有三种：
	1.	build 阶段依赖路径（如 rootdir / opensp include）：改 spec 里的宏或变量为你的 $PHYSROOT/sw/...
	2.	解包搬运阶段：不使用 <your user id>，用 home/*/*/sw/<pkg> 通配符定位真实 payload 路径
	3.	ocaml 这种运行时工具链：用环境变量 + wrapper 兜底（PATH/OCAMLLIB/OCAMLRUN/CAML_LD_LIBRARY_PATH + 过滤 -custom）

⸻

如果你愿意，我下一步可以把“我们已经成功的包”和“还没做的包”也列一张清单，并给出你接下来最可能踩的 5 类坑以及对应的预防操作（比如：每次 build 前自动检查 spec 是否含 /home/ilc/fujiik/sw、每次解包前自动 find 唯一目录等）。
