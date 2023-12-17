
1. 本地调试

安装ruby  https://rubyinstaller.org/downloads/  
需要一起安装devkit，记得添加环境变量
之后安装 Bundler 

``` bash
bundle install 
```

--macos只需要安装jekyll
gem安装没反应，需要修改镜像
gem sources
# 添加镜像源并移除默认源
gem sources --add https://mirrors.tuna.tsinghua.edu.cn/rubygems/ --remove https://rubygems.org/
# 列出已有源
gem sources -l
# 清空、更新缓存
gem sources -c
gem sources -u

<!-- gem env 查看gem path
gem install -n /Library/Ruby/Gems/2.6.0 jekyll -->
homebrew重新安装ruby然后覆盖环境变量

homebrew install ruby
gem install jekyll
export PATH="/opt/homebrew/opt/ruby/bin:$PATH"
export PATH="/opt/homebrew/lib/ruby/gems/3.2.0/bin:$PATH"
export LDFLAGS="-L/opt/homebrew/opt/ruby/lib"
export CPPFLAGS="-I/opt/homebrew/opt/ruby/include"

``` bash
# complie
cd E:\github\dimi-liu.github.io
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