# Fastlane使用手册——（一）Fastlane介绍及安装

## 1.介绍

正如[`fastlane`](https://fastlane.tools/)的官方介绍，`fastlane`是自动化部署beta版本和发布 `iOS` 和 `Android`应用程序的最简单方式。通过`fastlane`可以为我们节省大量的部署时间。本系列文章将用 `iOS` 为例。为读者全方位展示 `Fastlane` 的功能及使用中的注意事项。

文章环境如下:

```
Mac: macOS Mojave 10.14.6
ruby 2.6.3
fastlane 2.138.0
```

## 2.安装

1. 首先需要安装最新`Xcode command line tools`，在终端输入：

```
xcode-select --install
```
在安装完成后可以在次输入`xcode-select --instal`检查是否安装成功。

```
$ xcode-select --install 
// 提示已安装
xcode-select: error: command line tools are already installed, use "Software Update" to install updates
```

2.接下来安装`fastlane`。安装`fastlane`有两种安装方法，第一种使用`RubyGems`。第二种使用`Homebrew`。文章以 `RubyGems` 为例。`Homebrew`安装的可以自行搜索安装。

```
# 1.Using RubyGems
sudo gem install fastlane -NV
# 输入Mac密码

# 2.using Homebrew
brew cask install fastlane
```

### 2.1 系统 ruby 安装fastlane

因 Mac 电脑系统自带 ruby 环境，所以你可以直接执行安装代码 `sudo gem install fastlane -NV`,但直接使用系统安装可能会碰到一些权限的小问题，解决方案可参考下文 **问题**。
 笔者更推荐使用自己的`ruby`环境进行安装。（详见 2.2自定义ruby安装fastlane）。

### 2.2 非系统 ruby 安装 fastlane
我们可以使用`RVM`来管理、安装不同版本的 ruby，详见[https://ruby-china.org/wiki/rvm-guide](https://ruby-china.org/wiki/rvm-guide)。切换到自定义的 ruby 进行安装 fastlane 。

```
rvm安装 
curl -L get.rvm.io | bash -s stable  
安装成功后、启用rvm
source ~/.bashrc  
source ~/.bash_profile  
测试安装结果
rvm -v
查看ruby可安装版本
rvm list known
安装指定版本
rvm install 2.6.3(可自定义)
切换ruby
rvm use 2.6.3
设置rvm默认版本
rvm --default 2.6.3

安装fastlane:
sudo gem install fastlane -NV
```

安装完毕后，可以通过`fastlane -v`命令查看 fastlane 是否安装成功。

```
终端输入：
fastlane -v

fastlane installation at path:
/Users/xxx/.rvm/rubies/ruby-2.6.3/lib/ruby/gems/2.6.0/gems/fastlane-2.138.0/bin/fastlane
-----------------------------
[✔] 🚀 
fastlane 2.138.0
```

如上显示，证明 fastlane 安装成功。


### 问题:
使用gem安装可能会报错

- Connection reset错误

```
gem install xxx
ERROR:  While executing gem ... (Gem::RemoteFetcher::FetchError)
    Errno::ECONNRESET: Connection reset by peer - SSL_connect (https://api.rubygems.org/quick/Marshal.4.8/xxx.gemspec.rz)
```

解决：因为被墙的原因，我们需要切换`gem`的源,详见[RubyGems - Ruby China](https://gems.ruby-china.com)。
在终端中输入：


```
$ gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
$ gem sources -l
https://gems.ruby-china.com
# 确保只有 gems.ruby-china.com
```

- don't have write permissions，权限问题

```
ERROR:  While executing gem ... (Gem::FilePermissionError)
    You don't have write permissions into the /usr/bin directory.
```

解决：因为使用系统自带的`ruby`你是没有写的权限，可以执行下方安装代码：

```
sudo gem install fastlane -NV -n /usr/local/bin
```
