---
title: "使用 Alfred 切换 Network Location 和 Surge Profile"
date: 2018-09-07T20:53:45+08:00
tags: [Alfred, AppleScript, macOS, Network Location, Surge]
---

## 背景

忍不了公司的 Windows 电脑，所以自购了一台 Mac 带到公司去开发，同时也作为我的个人电脑使用。


### 几点实事

1. 在公司需要使用公司提供的 PAC；在家里在路由器上搭建了透明代理，本机不需要配置任何代理。
2. macOS 设置了 PAC 的情况下 Bypass proxy settings 将不再生效。
3. 公司提供的 PAC 行为不如我自己设置的 Rule-based Proxy 来的稳定，开发需要稳定的网络行为。
4. CLI 的代理设置切换通过 export 环境变量来设置，切换需要重开 iTerm Tab 或者在当前 Tab 内重新 source，并不方便。


### 解决方案

在公司：不使用公司提供的 PAC，使用 PAC 中的外网代理作为 Surge 的上游代理，设置 Surge 提供的代理为系统代理和 CLI 代理，通过 Surge 来决定请求是否走代理。

在家：系统不设置任何代理，CLI 代理（环境变量）仍然保持使用 Surge 的代理，Surge 的上游代理设置为 Shadowsocks。

<table>
  <tr>
    <th rowspan="2">Location</th>
    <th rowspan="2">System Proxy</th>
    <th rowspan="2">CLI Proxy</th>
    <th colspan="2">Surge Profile</th>
  </tr>
  <tr>
    <th>Upstream</th>
    <th>Rules</th>
  </tr>
  <tr>
    <td>Home</td>
    <td>None</td>
    <td>Surge</td>
    <td>Shadowsocks</td>
    <td>Daily rules</td>
  </tr>
  <tr>
    <td>Work</td>
    <td>Surge</td>
    <td>Surge</td>
    <td>HTTP proxy provided by my employer</td>
    <td>Working rules</td>
  </tr>
</table>


上述方法涉及到系统代理设置的切换可以通过 macOS 的 Network Location 功能来实现，并且有现成的 Alfred Workflow 可以使用，但是 Surge 切换则完全需要手动控制，这就忍不了了。所以目标是能够通过 Alfred 来控制 Surge。


## 通过 AppleScript 来操作 Surge

好在还有 AppleScript，查了一下 Surge 并没有 AppleScript 支持，但是最差的情况就是通过 AppleScript 来模拟人工操作来实现。只能选择通过 AppleScript 来点击 Surge 的 MenuBar item 来达到目的。

最终的 AppleScript 如下

```applescript
-- pass arguments to applescript
on run argv
	tell application "System Events"
		tell process "Surge"
			tell menu bar item 1 of menu bar 2
			-- "menu bar 2" is the right side of the entire menu bar, "menu bar item 1" is Surge's first menu bar item
				perform action "AXShowMenu"
				-- to show the menu
				tell menu item "Switch Profile" of menu 1 to {click, click menu item (item 1 of argv) of menu 1}
				-- switch profile according to the input argument
			end tell
			tell menu bar item 1 of menu bar 2
				perform action "AXShowMenu"
				keystroke "s" using command down
				-- Command + S, toggle "Set as System Proxy"
			end tell
		end tell
	end tell
end run
```

## 将切换 Network Location 的 Workflow 和这个 AppleScript 整合

我使用的切换 Network Location 的 Workflow 是[这个](http://www.packal.org/workflow/network-location)，这个 Workflow 是通过执行 bash 脚本来实现的，其中切换 Network Location 的功能是通过执行 Python 脚本来实现的。Python 部分完全不用改动，直接将我们写好的 AppleScript 放到这个 Workflow 的目录下面，然后在其执行的 bash 中添加一行:

`/usr/bin/osascript surge.applescript "{query}"`


这样并没有结束，`query` 变量是原来的 Workflow 的输入，该 Workflow 提供了选择 Network Location 的功能，不用自己手动输入 Network Location 的名字，所以这个 `query` 是要切换的目标 Network Location 的名字。在 AppleScript 中，是直接根据这个输入来切换 Profile 的，所以还需要将 Surge Profile 的名字和 Network Location 的名字统一。

## 其他

其实已经通过 Surge 来设置了系统的代理，所以 Network Location 的切换已经并不重要了，完全可以省去，但是我就是洁癖，rua。

## 参考

1. [用 AppleScript 操纵 Surge 菜单，提供 Keyboard Maestro 宏样例](https://github.com/jayqizone/Surge-AppleScript)
2. [一个 Surge 引发的血案--AppleScript从入门到放弃](https://x-front-team.github.io/2016/11/30/%E4%B8%80%E4%B8%AAsurge%E5%BC%95%E5%8F%91%E7%9A%84%E8%A1%80%E6%A1%88-AppleScript%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83/)


