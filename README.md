# Skinny Bones Jekyll Starter

Just a little something I'm using to jump start a site refresh. I like to think of it as a starter for building your own Jekyll site. I purposely keep the styling minimal and bare to make it easier to add your own flare and markup.

I'm currently using a variation of it on my personal website [Made Mistakes](http://mademistakes.com) with some modifications. To learn more about how to use the theme and install it check out the [Skinny Bones demo](http://mmistakes.github.io/skinny-bones-jekyll/) (*work in progress*).

![screenshot of Skinny Bones](http://mmistakes.github.io/skinny-bones-jekyll/images/skinny-bones-theme-feature.jpg)

---

## Notable Features

* Jekyll 3.x and GitHub Pages compatible.
* Stylesheet built using Sass.
* Data files for easier customization of the site navigation/footer and for supporting multiple authors.
* Optional Disqus comments, table of contents, social sharing links, and Google AdSense ads.
* And more.


ENTRYPOINT的作用是设置容器运行时的执行命令，即容器的默认入口
ENTRYPOINT有两种使用格式
    ENTRYPOINT ["executable", "param1", "param2"] (exec 格式, 有先)
    ENTRYPOINT command param1 param2 (shell 格式)

例如执行“docker run -i -t --rm -p 80:80 nginx”，命令中没有指定执行参数，即执行命令就是在镜像制作是，
	
使用shell格式会防止CMD或者run命令行参数被使用，但是有一个缺点，即ENTRYPOINT将以“/bin/sh -c”作为入口命令，
该命令不能传递信号量到子进程，也就是说用户指定执行实体不是作为容器的PID 1 来运行，且接受不到信号量，
因此当执行docker stop <container>命令时，执行实体接受不到SIGTERM信号

exec格式的ENTRYPOINT举例：
FROM ubuntu
ENTRYPOINT ["top", "-b"]
CMD ["-c"]

