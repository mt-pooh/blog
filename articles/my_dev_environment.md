# 3行まとめ
- え，エンジニアなのにシェルのカスタマイズしていなんっすか？
- WSL2 + WindowsTerminalがいい感じ
- Vimって楽しい

# シェル環境
## fishシェルを使う
POSIX非互換なのでたまに痛い目に遭うが，自動補完，シンタックスハイライトはなかなかよい。見た目も（デフォルトで）イケてるのでテンション上がりますよ。見た目にこだわるの大事。

最近はzsh + [starship](https://starship.rs/ja-jp/)に移行しようかと思っていたり・・・

## Viキーバインド
シェルのキーバインドはデフォルトではEmacsなので最近流行りのVimキーバインドに変更している。開発体験爆上がり。（最近まで知らなかった・・・）
カーソルを0で先頭に移動したり，bでを1ワード分前に移動させたりできる。感動ですよ。
Emacsよりもvimに馴染みがあるユーザーならシェル操作もviキーバインドで行うほうがいいよね。
- 切替方法
```fish
fish_vi_key_bindings
```
```bash
set -o vi
```
```zsh
bindkey -v
```
## alias よりも abbr (abbreviation) 
- alias はあくまで別の名前をつけるだけ
- abbr は略語であり、略してない正式名称がある、**実行時にコマンドが展開される**

例えば
```
# alias dco='docker-compose' 相当
abbr -a dco docker-compose
```
と設定すると`dco`と打つと`docker-compose`が展開されます。
<img width="1313" alt="abbr.gif (882.6 kB)" src="https://files.esa.io/uploads/production/attachments/5849/2020/10/24/78147/e974655a-bc86-403a-82e5-e829ccd3dd42.gif">

grep力が上がったり，人に見せるときにわかりやすかったりします。
fishには alias よりも abbrという思想があります。
zshでも[プラグイン導入](https://github.com/olets/zsh-abbr)で使えるようです。

私は展開してほしくない（完全に置き換える）コマンドはalias，展開してほしいコマンドはabbrというように使い分けています。
### my alias & abbr （一部）
```
# alias
alias ..='cd ..'
alias vi='nvim'
alias lg='lazygit'
alias relogin='exec $SHELL -l'
alias ls='exa'
alias ll='exa -ahl --git'

# abbr
abbr -a gst git status
abbr -a gc git commit
abbr -a gd git diff
abbr -a gca git commit --amend
abbr -a glo git log --oneline
abbr -a d docker
abbr -a dps docker ps
abbr -a dpa docker ps -a
abbr -a di docker images
abbr -a dex docker exec -i -t
abbr -a dco docker-compose
```
## 他にも
[fzf](https://github.com/junegunn/fzf): fuzzy finder
[ripgrep](https://github.com/BurntSushi/ripgrep): grepの進化版
[exa](https://github.com/ogham/exa): lsの進化版
[bat](https://github.com/sharkdp/bat): catの進化版
を導入してshellをかっこよく，快適にカスタマイズしています。

# WSL2 + Windows Terminal
最近のWindowsで開発するエンジニアはWSL2 + WindowsTerminalの組み合わせを使っている場合が多いのでは。
わたくしも例にもれず使っています。
WSL2を使うことでテスト実行などのプロセスが早くなったし，Windows Terminalでタブを作ったり，ウインドウズ分けれたりできて快適に開発できます。
Windows TerminalはMacの[iTerm2](https://www.iterm2.com/)になるべく近づけようとカスタマイズしています。（jsonファイルで管理）

- 背景は半透明に
- タスクバーに固定してWindows+1で起動するように
- ウインドウ分割で右側にlogを出している。
- 左の一番大きなウインドウでメインの作業を行う。

本当はvimのなかでterminal操作したいのだけど，慣れない・・・
# Vim
nvim使ってます。
パッケージは[dein](https://github.com/Shougo/dein.vim)で管理。
Language Server Protocolは[coc.nvim](https://github.com/neoclide/coc.nvim)で導入
typescriptも快適にかけます。補完や関数ジャンプも快適に行なえます。

VSCodeも[Vimキーバインド](https://github.com/VSCodeVim/Vim)でたまに使うけど，vscode側のキーバインドと衝突することがたまにあってうわーってなります。
Vimは自分色のテキストエディタに簡単にカスタマイズできるのでどんどん愛着湧いてきます。

閑話休題：
[Vimは使わなくていい、されどVimの思想を取り入れよ](https://note.com/navitime_tech/n/nf6cae399ae75)

# dotfileをgitで管理
以上の設定をgitで管理しています。新しい環境に移行する際にとても楽になります。
会社の開発環境とプライベートの開発環境を一緒にできるのもよい感じ。
https://github.com/mt-pooh/dotfiles

- `dotfilesLink.sh`を実行で設定ファイルのsymlinkを張り，いつものシェル環境+ vim環境ができあがる
- 環境独自の設定（社内プロジェクトでしか使わないaliasとか）はgit管理下に入れない

TODO:
symlink張る前に手作業で必要なパッケージのインストールが必要でこれを自動化したい。ansibleマンになるのかなぁ
�
