---
title: Setting Up a Decent Haskell Windows Environment 
date: "2019-09-25T22:12:03.284Z"
descriptions: We're cheating by using WSL, but that's okay.
---

0) Install VSCode

  - Set up some cool extensions:

   - https://marketplace.visualstudio.com/items?itemName=Shan.code-settings-sync to sync settings to Github

   - https://marketplace.visualstudio.com/items?itemName=CoenraadS.bracket-pair-colorizer for rainbow brackets

   - https://marketplace.visualstudio.com/items?itemName=wayou.vscode-todo-highlight to highlight todos

   - https://gitlens.amod.io/ to show git info

1) Set up WSL

2) Set up the new windows terminal

  - Get Ubuntu Terminal set up

  - Set up fonts

  - Set up zsh + oh-my-zsh + powerline10k

  - Set up git credential manager (https://www.edwardthomson.com/blog/git_credential_manager_with_windows_subsystem_for_linux.html) (also requires git for windows)

3) Install Stack

4) Install Haskell-Ide-Engine

  - remember to tell people to run `stack haddock --keep-going` for docs

5) install ghcid

6) Install VScode remote editing thing and set up the vscode hie extension

  - Set up vscode as git editor (`git config --global core.editor "code --wait"`)