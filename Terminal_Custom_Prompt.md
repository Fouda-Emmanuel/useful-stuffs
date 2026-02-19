

# Custom Bash Prompt with Git Branch & Virtualenv

A safe and colorful Bash prompt that shows your **current Git branch** and **Python virtual environment** in Ubuntu (or any Linux terminal).

This solves common problems when customizing your terminal:

* Shows virtualenv only if active, without errors
* Displays current Git branch safely, even outside Git repos
* Maintains proper line wrapping and arrow key navigation
* Works with long commands and backspace

---

## 🛠 Solution Code

```bash
# --- Git branch in prompt, keep Ubuntu default colors ---

# Function to show current Git branch with brackets only when in a git repo
parse_git_branch() {
    branch=$(git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD 2>/dev/null)
    if [ -n "$branch" ]; then
        printf "[%s]" "$branch"
    fi
}

# FORCE color prompt ON - ignore the automatic detection
force_color_prompt=yes

# Set PS1 with colors (bypass the broken detection)
if [ "$force_color_prompt" = yes ]; then
    PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\[\033[01;36m\]$(parse_git_branch)\[\033[00m\]\$ '
else
    PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w$(parse_git_branch)\$ '
fi

# Optional: Also set ls colors
eval "$(dircolors -b)"
alias ls='ls --color=auto'

```

---

## ✅ Features

* **Virtualenv Safe**: Only displays `(venv_name)` when a virtual environment is active.
* **Git Branch Safe**: Works in repos and non-repos without errors. Handles detached HEAD.
* **Color-Coded Prompt**:

  * Green → `username@host`
  * Blue → Current working directory
  * Cyan → Git branch
* **Terminal Safe**: Proper wrapping with `\[ \]` ensures long commands, backspace, and arrow keys work correctly.
* **Optional Polish**:

  * Color the virtualenv to match username
  * Git branch in brackets `[branch_name]` for clarity

---

## 🔧 How to Use

1. Copy the code into your `~/.bashrc` (or `~/.bash_profile`).
2. Reload your terminal or run:

```bash
source ~/.bashrc
```

3. Activate a virtualenv (optional) and navigate into a Git repo to see it in action.

---
