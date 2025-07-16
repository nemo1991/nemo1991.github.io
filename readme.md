# github pages roads

## step by step

1. create repo as username.github.io  
   如果要创建用户或组织站点，则存储库必须命名为 <user>.github.io 或 <organization>.github.io。 如果您的用户或组织名称包含大写字母，您必须小写字母
2. 发布源？ 
   从分支进行发布，关键就是选择 分支与目录
   使用github actions


install jekyll  
[install jekyll on mac](https://jekyllrb.com/docs/installation/macos/)  
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install chruby ruby-install
ruby-install ruby 3.4.1


echo "source $(brew --prefix)/opt/chruby/share/chruby/chruby.sh" >> ~/.zshrc
echo "source $(brew --prefix)/opt/chruby/share/chruby/auto.sh" >> ~/.zshrc
echo "chruby ruby-3.4.1" >> ~/.zshrc # run 'chruby' to see actual version


ruby -v

```
quick start  
[quick start](https://jekyllrb.com/docs/)  

```
gem install jekyll bundler
jekyll new myblog
cd myblog
bundle exec jekyll serve
```

切换ruby源

```
gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
```


https://blog.csdn.net/weixin_41287260/article/details/102582315

之前购买的域名解析CNAME删除后，依然会跳转到绑定的域名。
绑定新域名，删除浏览器缓存或使用隐私模式，好了。