# About

Setting up a Macbook all over again.


## System settings

- Settings -> Trackpad -> Point & Click -> Check box "Tap to click"
- Settings -> Accessibility -> Mouse & Trackpad -> Trackpad Options -> Check box "Enable dragging" with option "without drag lock"
- Settings -> Security & Privacy -> Firewall -> Turn on Firewall
- Settings -> Security & Privacy -> FileVault -> Turn on FileVault. Also connect charger to start encryption
- Settings -> Security & Privacy -> General -> Check box "Require password" IMMEDIATELY "after sleep or screen saver begins"


## Install basic software

- Use Safari to download Chrome
- Use Chrome to download Firefox
- Install Slack
- Install 1Password
- Install iTerm2
- Install Docker Community Edition
- Install Homebrew (DO NOT pipe to bash; download the script before running it)


## setup dotfiles

Edit `~/.bash_profile` to:

```
source ~/.bashrc
```

Then run the following:

```
git clone git://github.com/yanhan/dotfiles.git
git checkout mac
python setup_home_folder_dotfiles.py bashrc gitconfig tmux.conf vimrc zshrc
```

There will be errors here and there until you finish installing everything. In the meantime, just bear with it.


## Install software required for development

- Use homebrew to install pyenv: `brew install pyenv`
- Use homebrew to install pyenv-virtualenv: `brew install pyenv-virtualenv`
- Add the line `eval "$(pyenv virtualenv-init -)"` to `~/.bash_profile`
- Install Python 3.7.0: `pyenv install 3.7.0`.
- Clone https://github.com/yanhan/provision-mac and set it up to install other software we use
- Install nvm by following instructions at https://github.com/creationix/nvm#git-install . There is no need to add the code to `~/.bashrc` or similar if you have setup the dotfiles
- Use `nvm ls-remote` to find out what's available. Install an even numbered version. In late July 2018, it's `v10.7.0`
- Java 8: go to http://www.oracle.com/technetwork/java/javase/downloads/index.html and download and install accordingly


## tmux

Install tmux plugin manager:

```
git clone git://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

Then start a new tmux session and press `Prefix + I` for installation


## iTerm2 solarized

Clone the repo:

```
git clone git@github.com:altercation/solarized.git
```

On iTerm2, go to Preferences -> Profiles -> Colors tab -> CColor Presets dropdown -> Solarized Dark


## oh-my-zsh

First, change your shell to zsh

```
chsh -s /bin/zsh USERNAME
```

Then start a new shell, download the oh-my-zsh install script, go through it, then run it:

```
curl -fsSL -o 'install-omzsh'  'https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh'
chmod 700 ./install-omzsh
./install-omzsh
```

Not sure if we need the following:

```
cd ~/dotfiles
python setup_home_folder_dotfiles.py zshrc
```


## Install Stack

```
curl -fsSL -o 'install-stack'  'https://get.haskellstack.org/'
chmod 700 ./install-stack
./install-stack
```


## zsh-git-prompt

```
git clone git@github.com:olivierverdier/zsh-git-prompt.git
cd zsh-git-prompt
stack setup
stack build
stack install
```


## PostgreSQL

Follow the instructions [here](/install-postgresql-on-mac.md)
