---
title: 从 Vimium 到 qutebrowser
date: 2020-01-01 14:21:32
tags: [Vim,Browser,Equipment]
categories: Essay
---

过去两年，不论我安装、切换到哪家浏览器，~~除了已逝的 Vimperator~~，Vimium 都是第一个安装的插件。

曾经偶然听闻 qutebrowser 大名，但得知它没有让我难以割舍的 Dark Reader 插件，因此擦肩而过。

然而 Vimium 的小缺陷屡屡挑衅我的耐心，直到真正开始使用 qutebrowser，终于让我下定了迁移的决心。

<!--more-->

## 对比

Vimium 只是一个浏览器插件，Firefox 和 Chrome 均有支持，可以说在不破坏原有操作体验的同时，补充了一些键盘操作的效率提升。

可它的工作方式是页面注入式的，必须等当前页面完成初始加载后才能使用键盘操作，又有一些页面注定无法完成注入，例如浏览器自己的插件商店、配置页。

qutebrowser 则是一个 PyQt 实现的轻量 GUI 跨平台浏览器，默认基于 Chromium 内核，并专注于键盘操作。不论页面是否完成加载，都可以随时使用键盘做出强大快捷的操作。

可惜缺少插件系统，且对 inspector 的支持很差，Web 开发者们可能难以接受。

它们最大的相同点，当属两者的按键都是 Vim 风格。即使之前从未体验过的用户，看看键位图也能理解它们的差异了。

![vimium cheatsheet](/image/from-vimium-to-qutebrowser/vimium-cheatsheet.png)

![qutebrowser cheatsheet](/image/from-vimium-to-qutebrowser/qutebrowser-cheatsheet.png)

## 文档

qutebrowser 自带离线文档，`:help` 即可快捷查看，深度使用的话，自然要过上几遍。
接下来，我会介绍几个要点信息，帮助其他感兴趣的 Vimium 用户无痛切换到 qutebrowser。

## 配置文件

qutebrowser 的配置管理十分方便，支持通过修改文件自定配置。

不仅可以用 yml 文件做基础定义，还能使用 python 满足更多的定制需要。因而更推崇直接使用 config.py 做配置管理，Linux 平台在 `~/.config/qutebrowser/config.py`，Mac `~/.qutebrowser/config.py`， Windows 是 `%APPDATA%/qutebrowser/config/config.py`。

如果一开始配置文件不存在，可执行 `:config-write-py` 初始化。另有可选参数 `--force`，强制用当前配置覆写磁盘文件。

## 搜索引擎

qutebrowser 默认搜索引擎为 duckduckgo，但可以按需增加自己的常用配置。

```py
c.url.searchengines = {
    'DEFAULT': 'https://google.com/search?q={}',
    'google': 'https://google.com/search?q={}',
    'duckduckgo': 'https://duckduckgo.com/?q={}',
    'github': 'https://github.com/search?q={}',
    'npm': 'https://npmjs.com/search?q={}',
    'baidu': 'https://baidu.com/s?wd={}',
    'mijisou': 'https://mijisou.com/search?q={}'
}
```

## 键位迁移

作为 Vimium 老用户，我保留了之前的使用习惯，修改如下键位，并参考 [Amos Bird](https://github.com/amosbird/serverconfig) 的配置，增加了在模式转换时自动切换中文输入法状态。这里使用的是 fcitx-remote，在 Mac 下，可以借助 [fcitx-remote-for-osx](https://github.com/xcodebuild/fcitx-remote-for-osx) 实现同样的效果。

```py
# Bindings for normal mode
config.bind('x', 'tab-close')
config.bind('X', 'undo')
config.bind('J', 'tab-prev')
config.bind('K', 'tab-next')
config.bind('d', 'scroll-page 0 0.5')
config.bind('u', 'scroll-page 0 -0.5')
config.bind('j', 'scroll-page 0 0.1')
config.bind('k', 'scroll-page 0 -0.1')
config.bind('i', 'enter-mode insert ;; spawn fcitx-remote -t')
config.bind('gi', 'hint inputs --first ;; spawn fcitx-remote -t')
config.bind('p', 'open -- {clipboard}')
config.bind('P', 'open -t -- {clipboard}')
config.unbind('gl')
config.unbind('gr')
config.bind('gj', 'tab-move -')
config.bind('gk', 'tab-move +')
config.bind('<Escape>', c.bindings.default['normal']['<Escape>'] + ' ;; fake-key <Escape> ;; clear-messages ;; jseval --quiet document.getSelection().empty()')

# Bindings for insert mode
config.bind('<Ctrl-a>', 'fake-key <Home>', mode='insert')
config.bind('<Ctrl-e>', 'fake-key <End>', mode='insert')
config.bind('<Ctrl-d>', 'fake-key <Delete>', mode='insert')
config.bind('<Ctrl-h>', 'fake-key <Backspace>', mode='insert')
config.bind('<Ctrl-k>', 'fake-key <Ctrl-Shift-Right> ;; fake-key <Backspace>', mode='insert')
config.bind('<Ctrl-f>', 'fake-key <Right>', mode='insert')
config.bind('<Ctrl-b>', 'fake-key <Left>', mode='insert')
config.bind('<Ctrl-n>', 'fake-key <Down>', mode='insert')
config.bind('<Ctrl-p>', 'fake-key <Up>', mode='insert')
config.bind('<Escape>', 'spawn fcitx-remote -t ;; leave-mode ;; fake-key <Escape>', mode='insert')
```

## 代理配置

通过 `c.content.proxy`，可以轻松配置自己的代理，支持 http 和 socks 协议。

为了简化配置过程，我使用了 `Privoxy` 做了 http PAC 代理，可以参考我在 Mac 平台的 [Shell 脚本](https://github.com/Claude-Ray/dotfiles/blob/master/macos/privoxy.sh)。

## 窗口最大化

绝大部分时间，我的浏览器是处于窗口最大化的。当然不是 Mac 原生的全屏，私以为那种另开一个桌面的全屏模式体验太差，不仅窗口切换动画时间长，也无法与别的任务窗口叠加，总有需要抄点东西的时候。

在 Mac 上，qutebrowser 的 title bar 实在是又丑又大，可以通过 `c.window.hide_decoration = True` 来关闭它。但至今还存在的一个问题是关闭 title bar 之后，无法再调整窗口大小。

即使在更改设置之前 qutebrowser 窗口处于最大化状态，hide_decoration 只能起到隐藏 title bar 的效果，体现到界面上就是残缺的一条空白。身为强迫症简直不能忍！

一番琢磨，终于找到了临时的解决办法：

1. 先把 hide_decoration 关掉，在浏览器上快捷执行 `:set window.hide_decoration false`
2. 再将浏览器全屏，对应指令 `:fullscreen`
3. 执行 `:set window.hide_decoration true`
4. 按 `Ctrl+Up` 或用手势操作进入 Mission Control 界面，将最大化的 qutebrowser 从新桌面中拖拽到原来的桌面
5. 这时 qutebrowser 进入短暂的“无响应”阶段，用鼠标点击或滚动一下窗口的任意地方即可重新激活

这样就获得了无边框最大化的 qutebrowser。经过检验，重启 qutebrowser，甚至重启系统之后均能保持窗口最大化。

我还加了 title bar 的热键简化操作（下面 Meta 实为 Command/Super）：

```py
c.window.hide_decoration = True

config.bind('<Meta-Ctrl-f>', 'config-cycle window.hide_decoration false true')
```

## 总结

经过一个多星期的使用，qutebrowser 流畅的操作体验令我开怀不矣，现在彻底将它作为主力浏览器。

另一方面，它支持 insert 模式的按键定制，可以让我们在其他系统环境下，像 Mac 一样在浏览器中使用 Emacs-like 键位做行内编辑！

假如你之前从未使用过 Vim-like 的浏览器插件，可以先把 Vimium 装起来。即使键位需要一点时间适应，可它胜在有着极大的包容性——不存在模式切换，没有按键冲突，也就不用担心它会降低你既有的操作效率。

最后，我的配置都在自己的 [dotfiles](https://github.com/Claude-Ray/dotfiles) 仓库，希望对你有所帮助。