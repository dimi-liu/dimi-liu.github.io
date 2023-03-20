
1. 本地调试

安装ruby  https://rubyinstaller.org/downloads/  
需要一起安装devkit，记得添加环境变量
之后安装 Bundler 

``` bash
bundle install 
```

``` bash
# complie
jekyll serve
# Server address: http://127.0.0.1:4000/
```

1. 生成新的markdown

```bash
    rake post title="main" subtitle="sub"
```

2. 提交格式

    post + 内容

3. 修改渲染代码

    在layout里面，一些引用在include，自定义了mathjax类，数学公式