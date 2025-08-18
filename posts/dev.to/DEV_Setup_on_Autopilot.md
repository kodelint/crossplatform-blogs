---
title: Dev Setup on Autopilot
cover_image: 'https://raw.githubusercontent.com/kodelint/kodelint.github.io/refs/heads/main/assets/uploads/01-setup-devbox.png'
description: null
tags: 'rust, macOS, productivity, cli'
series: null
canonical_url: null
published: null
id: 2781690
---

## Dev Setup on Autopilot

Every developer knows the ritual. A new laptop arrives shiny, fast, and brimming with potential. Or perhaps you’re upgrading your existing machine, or starting a new role with a fresh corporate setup. The excitement is palpable, but then reality hits: the painstaking process of recreating your perfect development environment.

For years, I’ve lived this saga. Like many of you, I have my carefully curated arsenal of tools, fonts, and configurations that make me productive. My terminal needs to look just right, my version managers must be in place, and a dozen specific CLI utilities are non-negotiable.

## The Deja Vu of Re-Setup

The problem isn’t just the initial setup; it’s the **memory** part. I’d spend hours, sometimes days, painstakingly recalling, searching, and reinstalling. Let me give you some concrete examples of the frustration:

1. **The Elusive brew Formula:** You remember using `brew` to install that niche CLI tool for managing cloud resources, but was it brew install `aws-cli` or `brew install awscli`? Did it have a specific `--with-zsh-completion` flag? Or was it `aws-cli@2` because the latest version broke a script? You spend 15 minutes in brew search and then another 10 digging through old documentation or `~/.zsh_history` (if you even remembered to back it up).

2. **The Go Tooling Conundrum:** You need `protoc-gen-go-grpc`. Did you `go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest`? Or was it an older `v1.2.3` because the team's build system isn't on the newest `protobuf`? Or, even worse, did you download a pre-compiled binary from a **GitHub Release** because it was faster, and now you have no idea where it came from?

3. **The Perfect Font, Gone:** Your beautiful coding font, say **“Fira Code Nerd Font Mono”** suddenly isn’t rendering correctly. Was it version `v3.0.2` or `v3.1.1` of **Nerd Fonts**? And which specific `*.ttf` file within the archive did you copy to `~/Library/Fonts`? Now you're hunting through **GitHub Releases**, downloading multi-gigabyte archives just to find that one precise file.

4. **Dotfiles’ Blind Spots:** While a `~/.dotfiles` repository is a lifesaver for shell aliases (`alias gcm='git commit -m'`) or Vim plugins, they notoriously fall short when it comes to:
   - **Actual Software Installation:** Your dotfiles won’t magically install Homebrew, or `git` itself, or `Node.js`, or the `Rust` toolchain. They assume these foundational tools are already there.
   - **Installation Methods:** Your `shellrc` might expect `pyenv` to be in your `$PATH`, but it won't tell you if `pyenv` was installed via `brew`, `git clone`, or a custom script.
   - **System-Level Settings:** Those **macOS** `defaults write` commands that perfect your trackpad speed, hide desktop icons, or ensure hidden files are always visible, these are often scattered notes or manual tweaks, not cleanly managed.

This constant “re-setup tax” was a major pain point, a recurring nightmare that chipped away at my valuable time and focus. I wanted a way to truly encapsulate my **entire** development environment, not just bits and pieces.

---

### My Solution: Introducing `setup-devbox`

That frustration finally boiled over, and I decided to tackle it head-on. I focused on a piece of software that solves this exact problem: [`setup-devbox`](https://github.com/kodelint/setup-devbox).

[`setup-devbox`](https://github.com/kodelint/setup-devbox) is my answer to the "new laptop dilemma." It’s a command-line tool that lets you define your entire development environment in **declarative `YAML` files**. This isn't just about listing tools; it's about stating the **desired end-state** of your machine, which is repeatable every single time, in a format that's both human-readable and machine-executable.

#### Why Declarative `YAML`?

The power of declarative YAML is that it acts as your single source of truth. It lives alongside your code in version control, meaning:

- **Readability:** It’s easy to glance at `tools.yaml` and instantly know what's supposed to be installed.
- **Version Control:** You can track changes to your environment setup over time, just like you track changes to your application code.
- **Shareability:** Want to onboard a new team member to an identical setup? Share your [`setup-devbox`](https://github.com/kodelint/setup-devbox) config files.
- **No More Guesswork:** No more trying to remember a specific brew flag or the exact go install command. The `YAML` specifies it all.

#### What [`setup-devbox`](https://github.com/kodelint/setup-devbox) Manages:

When you define your environment, [`setup-devbox`](https://github.com/kodelint/setup-devbox) steps up to handle the heavy lifting across several crucial domains:

- **Tools (and Their Installation Methods!):** This is where it shines. You define tools like `git`, `Node.js`, `kubectl`, or even specific versions of `gh` (GitHub CLI). Crucially, you also tell [`setup-devbox`](https://github.com/kodelint/setup-devbox) how to install them: - `brew`: For standard packages on `macOS` and `Linux`. - `cargo`: For your favorite Rust CLI tools. - `github`: For tools distributed as **GitHub Releases** (e.g., specific versions of `terraform` or `helm`). - `url`: For direct downloads of binaries or installers that don't fit other categories (e.g., a specific Go installer `.pkg`).
  It intelligently uses the correct installer and even handles `rename_to` for convenience (e.g., `cli` to `gh`).

- **macOS System Settings (`defaults write`):** Tired of manually changing Finder settings or keyboard behaviors after a clean install? [`setup-devbox`](https://github.com/kodelint/setup-devbox) lets you declare those `defaults write` commands directly in your `settings.yaml`, ensuring consistency every time.

- **Shell Configuration (`.zshrc`, `.bashrc` etc.):** Beyond just aliases, you can define raw lines of configuration to be injected into your shell startup files. This includes everything from `$PATH` modifications to eval "`$(starship init zsh)`" or complex `fzf` bindings.

- **Fonts:** My coding experience isn’t complete without specific fonts, often Nerd Fonts. [`setup-devbox`](https://github.com/kodelint/setup-devbox) automates the download and installation of these, ensuring my terminal looks perfect on every machine.

At present it support **GitHub, Brew, Cargo, Rustup, Go, Pip** and **Direct URL**.

**Here are examples of my `tools.yaml`**

```yaml
- name: git-spellcheck
  version: 0.0.1
  source: github
  repo: kodelint/git-spellcheck
  tag: v0.0.1
  rename_to: git-spellcheck
- name: git-pr
  version: 0.1.0
  source: github
  repo: kodelint/git-pr
  tag: v0.1.0
  rename_to: git-pr
- name: cli
  version: "2.74.0"
  source: github
  repo: cli/cli
  tag: v2.74.0
  rename_to: gh
- name: delta
  version: "0.18.2"
  source: github
  repo: dandavison/delta
  tag: 0.18.2
  rename_to: delta
```

**Example of `fonts.yaml`**

```yaml
fonts:
- name: 0xProto
  version: 3.4.0
  source: github
  repo: ryanoasis/nerd-fonts
  tag: v3.4.0
  install_only: ["regular", "Mono"]
  Example of shellrc.yaml
```

**Example of `shellrc.yaml`**

```yaml
shellrc:
  shell: zsh
  raw_configs:
   - export PATH=$HOME/bin:$PATH
   - export PATH=$HOME/.cargo/bin:$PATH
   - export PATH=$HOME/go/bin:$PATH
   - eval "$(starship init zsh)"
   - "[ -f ~/.fzf.zsh ] && source ~/.fzf.zsh"
   - zle -N fzf-history-widget - bindkey '^R' fzf-history-widget
   - 'export FZF_CTRL_R_OPTS="--no-preview --scheme=history --tiebreak=index   --bind=ctrl-r:toggle-sort"'
   - | export FZF_CTRL_T_OPTS="
      --preview 'bat --style=numbers --color=always {} || cat {}'
      --preview-window=right:80%
      --bind ctrl-b:preview-page-up,ctrl-f:preview-page-down"
aliases:
    - name: code
      value: cd $HOME/Documents/github/
    - name: gocode
      value: cd $HOME/go/src/
    - name: edconf
      value: zed $HOME/.zshrc
    - name: g
      value: git
    - name: gs
      value: g status
```

**Example of `settings.yaml`**

```yaml
settings:
  macos:
    - domain: NSGlobalDomain
      key: AppleShowAllExtensions
      value: "true"
      type: bool
    - domain: com.apple.finder
      key: AppleShowAllFiles
      value: "true"
      type: bool
    - domain: com.apple.dock
      key: expose-animation-duration
      value: "0.1"
      type: float
    - domain: NSGlobalDomain
      key: KeyRepeat
      value: "1"
      type: int
    - domain: com.apple.finder
      key: ShowPathbar
      value: "true"
      type: bool
```

#### The Magic of State Management

The core idea is simple yet powerful: you run `setup-devbox now`, and it springs into action. But it's not a dumb script that reinstalls everything every time. It leverages an internal `state.json` file. This file acts as `setup-devbox`'s memory, keeping track of what has already been installed and configured, and how.

This means that:

- **Idempotency:** Running `setup-devbox` now multiple times won't break anything or perform redundant work. If a tool is already installed and matches your desired version, `setup-devbox` simply skips it.

- **Efficiency:** Only missing or outdated components are installed or updated, saving you precious time and bandwidth.

- **Reliability:** The `state.json` ensures that your environment progresses predictably towards the desired state you've defined.

This means that whether I’m upgrading my MacBook, spinning up a new virtual machine, or onboarding onto a client’s specific environment, I can replicate my ideal setup with minimal effort. The entire “re-setup tax” is almost eliminated, freeing me up to do what I love: **develop**.

### Reclaim Your Setup Time, Forever

Building [`setup-devbox`](https://github.com/kodelint/setup-devbox) has transformed my personal development workflow from a recurring chore into a one-command automation. My development environment is no longer a collection of scattered memories and manual steps; it’s a declarative, version-controlled blueprint that setup-devbox brings to life with intelligent efficiency.

If you, like me, have suffered the “**new laptop setup**” headache too many times, I genuinely believe setup-devbox can be a game-changer for you. It's my personal solution to a universal developer problem, and I'm excited to share it with the community.

Head over to the GitHub Repository: [`setup-devbox`](https://github.com/kodelint/setup-devbox), explore the configuration examples, and give [`setup-devbox`](https://github.com/kodelint/setup-devbox) a try. I'm actively developing it, and your feedback, ideas, or even contributions would be incredibly valuable.

#### Repository: [`setup-devbox`](https://github.com/kodelint/setup-devbox)

#### Happy Developing!!
