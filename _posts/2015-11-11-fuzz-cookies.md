---
layout: post
title: 利用高校教务系统cookie漏洞进行Fuzz爆破
date: 2015-11-11 16:22:34
categories: 网络安全
permalink: /archivers/fuzz_cookie
---
<style type="text/css">
	.indent{
		text-indent:2em;
	}
</style>
<p class="indent">没读懂标题的朋友先出门左拐百度/google一下fuzz和cookie的意思，不谢。</p>
<p class="indent">本文将分析作者所在大学教务系统对于cookie的设计逻辑中不严谨的地方，获取并利用漏洞，最后分析这个漏洞的价值（危害性）。首先需要说明的是，漏洞本身并没有什么高深的地方，当然这也可能是漏洞一直存在没有被打补丁的原因。因此，我努力希望能够让不同基础的人都能读懂。</p>
<p class="indent">实际上，每一个对http消息有所了解同学都能够很轻松的发现这个漏洞。那么我们直接切入核心，对登录教务系统阶段的跳转链接使用<a href="http://drops.wooyun.org/tools/1548">burpsuite</a>抓取http消息。那么需要明确的是，登录教务系统的过程包括两条消息，其中一条为POST，另一条为GET。在了解了cookie的基础上我们可以明白，POST消息中request请求中写入学号（stuid）和口令（pwd）并递交表单，返回的response中包含了对我们<strong>十分重要</strong>的cookie值；在GET消息中，request请求填入cookie值，返回的response中包含登录反馈的消息。那么由此可以显然发现一些猫腻。也就是说，如果可以得到一个登录用户的cookie，那么就可以访问这个用户的教务系统。道理是这样讲没错...</p>
<p class="indent">那么问题来到了“获取cookie”这里。做过后台的同学应该会很清楚，cookie由服务器端分配，作为凭证保存用户信息。通过分析登录成功后的提示消息（“<i>离开时请注销</i>”&“<i>每次登录时间限为30分钟</i>”）可以大概猜测到，服务器为登录成功的用户分cookie，这个cookie的存活周期为30min，并且在用户注销登录时会销毁这个cookie。那么cookie的庐山真面目到底是什么样子的呢？现在是时候放一波图片了...</p>
<p><img src="/img/f1.png" /></p>
<p><img src="/img/f2.png" /></p>
<p class="indent">从这两张图片的信息中我们可以发现cookie的一个大问题是过于简单！cookie是学号+时间戳的一串纯数字，也就是说我们可以通过fuzz的方法进行暴力破解。为了证明这种方法的有效性，很荣幸的请到了Wang同学来充当演员。（tips：我已经知道了何同学的学号。）破解结果如下：</p>
<p><img src="/img/f3.png" /></p>
<p><img src="/img/f4.png" /></p>
<p class="indent">现在就可以在Wang同学的教务系统里随便转了。</p>
<p class="indent">当对整个漏洞有了充分的掌握之后回过头去反思这个事件时，实际上漏洞的危害性并不是很大。因为漏洞被利用的条件相对苛刻，要求在受害者登录系统30min内猜解出cookie，并且还要求已知受害者的学号。当然学号与口令的匹配也可以进行划定范围进行模糊测试，但是这样一来运算的时间复杂度会呈现爆炸式的增长。所以如果想更好利用这个漏洞进行渗透，那么结合社会工程学的知识和其他更容易得到的库（例如学号库）进行弱口令猜解也会是一个值得尝试的方向。</p>
<p class="indent">所以你害怕你选不到舍友选上的课么？</p>
