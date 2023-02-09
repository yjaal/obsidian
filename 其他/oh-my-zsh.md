
```shell
# If you come from bash you might have to change your $PATH.
# export PATH=$HOME/bin:/usr/local/bin:$PATH

# Path to your oh-my-zsh installation.
export ZSH=$HOME/ohmyzsh

# Set name of the theme to load --- if set to "random", it will
# load a random theme each time oh-my-zsh is loaded, in which case,
# to know which specific one was loaded, run: echo $RANDOM_THEME
# See https://github.com/ohmyzsh/ohmyzsh/wiki/Themes
ZSH_THEME="robbyrussell"

# Set list of themes to pick from when loading at random
# Setting this variable when ZSH_THEME=random will cause zsh to load
# a theme from this variable instead of looking in $ZSH/themes/
# If set to an empty array, this variable will have no effect.
# ZSH_THEME_RANDOM_CANDIDATES=( "robbyrussell" "agnoster" )

# Uncomment the following line to use case-sensitive completion.
# CASE_SENSITIVE="true"

# Uncomment the following line to use hyphen-insensitive completion.
# Case-sensitive completion must be off. _ and - will be interchangeable.
# HYPHEN_INSENSITIVE="true"

# Uncomment the following line to disable bi-weekly auto-update checks.
# DISABLE_AUTO_UPDATE="true"

# Uncomment the following line to automatically update without prompting.
# DISABLE_UPDATE_PROMPT="true"

# Uncomment the following line to change how often to auto-update (in days).
# export UPDATE_ZSH_DAYS=13

# Uncomment the following line if pasting URLs and other text is messed up.
# DISABLE_MAGIC_FUNCTIONS="true"

# Uncomment the following line to disable colors in ls.
# DISABLE_LS_COLORS="true"

# Uncomment the following line to disable auto-setting terminal title.
# DISABLE_AUTO_TITLE="true"

# Uncomment the following line to enable command auto-correction.
# ENABLE_CORRECTION="true"

# Uncomment the following line to display red dots whilst waiting for completion.
# COMPLETION_WAITING_DOTS="true"

# Uncomment the following line if you want to disable marking untracked files
# under VCS as dirty. This makes repository status check for large repositories
# much, much faster.
# DISABLE_UNTRACKED_FILES_DIRTY="true"

"~/.zshrc" 147L, 4736B
# users are encouraged to define aliases within the ZSH_CUSTOM folder.
# If you come from bash you might have to change your $PATH.
# export PATH=$HOME/bin:/usr/local/bin:$PATH

# Path to your oh-my-zsh installation.
export ZSH=$HOME/.oh-my-zsh

# Set name of the theme to load --- if set to "random", it will
# load a random theme each time oh-my-zsh is loaded, in which case,
# to know which specific one was loaded, run: echo $RANDOM_THEME
# See https://github.com/ohmyzsh/ohmyzsh/wiki/Themes
ZSH_THEME="robbyrussell"

# Set list of themes to pick from when loading at random
# Setting this variable when ZSH_THEME=random will cause zsh to load
# a theme from this variable instead of looking in $ZSH/themes/
# If set to an empty array, this variable will have no effect.
# ZSH_THEME_RANDOM_CANDIDATES=( "robbyrussell" "agnoster" )

# Uncomment the following line to use case-sensitive completion.
# CASE_SENSITIVE="true"

# Uncomment the following line to use hyphen-insensitive completion.
# Case-sensitive completion must be off. _ and - will be interchangeable.
# HYPHEN_INSENSITIVE="true"

# Uncomment the following line to disable bi-weekly auto-update checks.
# DISABLE_AUTO_UPDATE="true"

# Uncomment the following line to automatically update without prompting.
# DISABLE_UPDATE_PROMPT="true"

# Uncomment the following line to change how often to auto-update (in days).
# export UPDATE_ZSH_DAYS=13

# Uncomment the following line if pasting URLs and other text is messed up.
# DISABLE_MAGIC_FUNCTIONS="true"

# Uncomment the following line to disable colors in ls.
# DISABLE_LS_COLORS="true"

# Uncomment the following line to disable auto-setting terminal title.
# DISABLE_AUTO_TITLE="true"

# Uncomment the following line to enable command auto-correction.
# ENABLE_CORRECTION="true"

# Uncomment the following line to display red dots whilst waiting for completion.
# COMPLETION_WAITING_DOTS="true"

# Uncomment the following line if you want to disable marking untracked files
# under VCS as dirty. This makes repository status check for large repositories
# much, much faster.
# DISABLE_UNTRACKED_FILES_DIRTY="true"
Last login: Wed Mar  2 20:35:37 on ttys001
[oh-my-zsh] Would you like to update? [Y/n] y
Updating Oh My Zsh
fatal: remote error:
  The unauthenticated git protocol on port 9418 is no longer supported.
Please see https://github.blog/2021-09-01-improving-git-protocol-security-github/ for more information.
There was an error updating. Try again later?
/Users/YJ/.zshrc:144: bad assignment
➜  ~ vim ~/.zshrc
➜  ~ source ~/.zshrc
[oh-my-zsh] Would you like to update? [Y/n] y
Updating Oh My Zsh
fatal: remote error:
  The unauthenticated git protocol on port 9418 is no longer supported.
Please see https://github.blog/2021-09-01-improving-git-protocol-security-github/ for more information.
There was an error updating. Try again later?
➜  ~ whereis oh-my-zsh
➜  ~ source ~/.zshrc
[oh-my-zsh] Would you like to update? [Y/n] y
Updating Oh My Zsh
fatal: remote error:
  The unauthenticated git protocol on port 9418 is no longer supported.
Please see https://github.blog/2021-09-01-improving-git-protocol-security-github/ for more information.
There was an error updating. Try again later?
➜  ~ vim ~/.zshrc
➜  ~ ls
Applications               Library                    Public                     softInstall.txt
Desktop                    Movies                     Virtual Machines.localized store
Documents                  Music                      logs                       study
Downloads                  Pictures                   soft
➜  ~ cd $ZSH
➜  .oh-my-zsh git:(master) ls
CODE_OF_CONDUCT.md README.md          custom             oh-my-zsh.sh       themes
CONTRIBUTING.md    SECURITY.md        lib                plugins            tools
LICENSE.txt        cache              log                templates
➜  .oh-my-zsh git:(master) pwd
/Users/YJ/.oh-my-zsh
➜  .oh-my-zsh git:(master) cd /Users/YJ
➜  ~ ls
Applications               Library                    Public                     softInstall.txt
Desktop                    Movies                     Virtual Machines.localized store
Documents                  Music                      logs                       study
Downloads                  Pictures                   soft
➜  ~ cd .oh-my-zsh
➜  .oh-my-zsh git:(master) ls
CODE_OF_CONDUCT.md README.md          custom             oh-my-zsh.sh       themes
CONTRIBUTING.md    SECURITY.md        lib                plugins            tools
LICENSE.txt        cache              log                templates
➜  .oh-my-zsh git:(master) git pull
hint: Pulling without specifying how to reconcile divergent branches is
hint: discouraged. You can squelch this message by running one of the following
hint: commands sometime before your next pull:
hint:
hint:   git config pull.rebase false  # merge (the default strategy)
hint:   git config pull.rebase true   # rebase
hint:   git config pull.ff only       # fast-forward only
hint:
hint: You can replace "git config" with "git config --global" to set a default
hint: preference for all repositories. You can also pass --rebase, --no-rebase,
hint: or --ff-only on the command line to override the configured default per
hint: invocation.
fatal: remote error:
  The unauthenticated git protocol on port 9418 is no longer supported.
Please see https://github.blog/2021-09-01-improving-git-protocol-security-github/ for more information.
➜  .oh-my-zsh git:(master) cd ..
➜  ~
# If you come from bash you might have to change your $PATH.
# export PATH=$HOME/bin:/usr/local/bin:$PATH

# Path to your oh-my-zsh installation.
export ZSH=$HOME/.oh-my-zsh

# Set name of the theme to load --- if set to "random", it will
# load a random theme each time oh-my-zsh is loaded, in which case,
# to know which specific one was loaded, run: echo $RANDOM_THEME
# See https://github.com/ohmyzsh/ohmyzsh/wiki/Themes
ZSH_THEME="robbyrussell"

# Set list of themes to pick from when loading at random
# Setting this variable when ZSH_THEME=random will cause zsh to load
# a theme from this variable instead of looking in $ZSH/themes/
# If set to an empty array, this variable will have no effect.
# ZSH_THEME_RANDOM_CANDIDATES=( "robbyrussell" "agnoster" )

# Uncomment the following line to use case-sensitive completion.
# CASE_SENSITIVE="true"

# Uncomment the following line to use hyphen-insensitive completion.
# Case-sensitive completion must be off. _ and - will be interchangeable.
# HYPHEN_INSENSITIVE="true"

# Uncomment the following line to disable bi-weekly auto-update checks.
# DISABLE_AUTO_UPDATE="true"

# Uncomment the following line to automatically update without prompting.
# DISABLE_UPDATE_PROMPT="true"

# Uncomment the following line to change how often to auto-update (in days).
# export UPDATE_ZSH_DAYS=13

# Uncomment the following line if pasting URLs and other text is messed up.
# DISABLE_MAGIC_FUNCTIONS="true"

# Uncomment the following line to disable colors in ls.
# DISABLE_LS_COLORS="true"

# Uncomment the following line to disable auto-setting terminal title.
# DISABLE_AUTO_TITLE="true"

# Uncomment the following line to enable command auto-correction.
# ENABLE_CORRECTION="true"

# Uncomment the following line to display red dots whilst waiting for completion.
# COMPLETION_WAITING_DOTS="true"

# Uncomment the following line if you want to disable marking untracked files
# under VCS as dirty. This makes repository status check for large repositories
# much, much faster.
# DISABLE_UNTRACKED_FILES_DIRTY="true"

"~/.zshrc" 147L, 4740B
# users are encouraged to define aliases within the ZSH_CUSTOM folder.
# For a full list of active aliases, run `alias`.
#
# Example aliases
# alias zshconfig="mate ~/.zshrc"
# alias ohmyzsh="mate ~/.oh-my-zsh"



###### my settings ########

# jdk
JAVA_HOME=/Users/YJ/soft/jdk8/Contents/Home
# JAVA_HOME=/Users/YJ/soft/jdk11/Contents/Home
PATH=$JAVA_HOME/bin:$PATH:.
CLASSPATH=$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:.
export JAVA_HOME
export PATH
export CLASSPATH

# maven
export M2_HOME=/Users/YJ/soft/maven3.6.3
export PATH=$PATH:$M2_HOME/bin

# gradle
GRADLE_HOME=/Users/YJ/soft/gradle-6.7
export GRADLE_HOME
export PATH=$PATH:$GRADLE_HOME/bin

# 抽风了，需要加一下
PATH=/bin:/usr/bin:/usr/local/bin:${PATH}
export PATH

# 设置vscode启动的命令别名
alias code="/Applications/Visual\ Studio\ Code.app/Contents/Resources/app/bin/code"

# golang
# 参考：https://studygolang.com/articles/11081
export GOPATH=/usr/local/Cellar/go/1.16.3
export PATH=$PATH:$GOPATH/bin
export GOBIN=/usr/local/Cellar/go/1.16.3/bin
export PATH=$PATH:$GOBIN
export GOROOT=/usr/local/Cellar/go/1.16.3/libexec
export PATH=$PATH:$GOROOT/bin
export GO111MODULE=on

alias vscode="/Applications/Visual\ Studio\ Code.app/Contents/Resources/app/bin/code"
alias python=/usr/bin/python3

# python selenium chrome 驱动
# export PY_SELENIUM_DRIVER = /Users/YJ/study/python-work/driver
# export PATH=$PATH:$PY_SELENIUM_DRIVER


# zookeeper
export ZK_HOME=/Users/YJ/soft/zk-3.7.0
export PATH=$PATH:$ZK_HOME/bin
```