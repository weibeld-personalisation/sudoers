# sudoers

Custom sudoers files for different platforms.

> Currently, only a generic Linux version is provided (known to work on Debian/Ubuntu). A version for macOS is planned).

## Introduction

This repository contains sudoers files for different platforms. A sudoers file defines the behaviour of the `sudo` command.

Every system has a default sudoers file named `/etc/sudoers`. This file contains all the default sudoers settings and it has an include directive for all files in the `/etc/sudoers.d` directory. That means, all the sudoers files in the `/etc/sudoers.d` directory will be applied along with the main `/etc/sudoers` file.

The sudoers files in this repository are intended to be put in the `/etc/sudoers.d` directory. They take into account the default `/etc/sudoers` file on the corresponding system and are designed to be applied on top of these default settings.

> Note that settings in `/etc/sudoers.d` files override the settings in the `/etc/sudoers` file.

## Features

The sudoers files in this repository largely implement the following common behaviour:

1. Password-less `sudo` for the default user
   - This means that this user can use `sudo` without entering a password
1. The `PATH` variable is passed to the `sudo` environment
   - This means that commands executed with `sudo` have the same `PATH` as the default user
1. The `HOME` variable is passed to the `sudo` environment
   - This means that commands executed with `sudo` use config files from the default user's home directory, such as `~/.vimrc`, `~/.bashrc`, etc.
1. A few additional common variables are passed to the `sudo` environment
   - Including: `EDITOR`, `http_proxy`, `https_proxy`, `no_proxy`

## Installation

To install the sudoers file, run the following as the default user:

```bash
curl https://raw.githubusercontent.com/weibeld/sudoers/main/linux | DATE=$(date -Iseconds) envsubst | sudo tee /etc/sudoers.d/config >/dev/null
```

The above saves the sudoers file for Linux as `/etc/sudoers.d/config` on the local machine.

Note the following about this command:

- The user for which password-less `sudo` is enabled is the user who executes the above command.
- The `envsubst` command is needed to replace the `$USER` placeholder in the downloaded file with the value of the `USER` environment variable on the local system.
  - `envsubst` is installed by default on most systems, if it isn't, it can be installed through the `gettext` package.
- The `DATE` variable assignment is optional and causes the current date to be included as a comment in created sudoers file.

## Notes

### HOME variable

One of the most crucial settings in a sudoers file is `env_keep += HOME` which causes the `HOME` variable (as well as the `~` character) in the `sudo` environment to be set to the invoking user's `HOME` variable instead of the root user's `HOME` variable.

The consequence of this is that commands executed in the `sudo` environment will use configuration files from the invoking user's home directory instead of the root user's home directory. For example, `sudo vim` will use the invoking user's `.vimrc` file and `.vim` directory, with all the familiar configurations, rather than the root user's version of these files (which are likely even non-existent).

This has also effects when an interactive root shell is started with `sudo -s` (not `sudo -i` as explained below). In this case, the started shell uses the invoking user's `.bashrc` file, including all the customisations, shell alias, shell functions, environment variables, etc., rather than the root user's `.bashrc` file.

> On macOS, `env_keep += HOME` is even included in the default `/etc/sudoers` file.

### Ways to invoke sudo

There are various ways to use `sudo`, the most important ones are listed below.

#### `sudo CMD`

- This runs a single command in a new environment that's defined by the settings int the `sudoers` file.
- It does **not** source any `.bashrc`, `.bash_profile` or any other config files.
- `CMD` must be an executable and **not** a shell alias, a shell function, or a shell builtin (`command not found` error otherwise).

> There's a trick to create the illusion that `sudo` can actually execute Bash aliases by defining the following:
>  ```bash
>  alias sudo='sudo '
>  ```
>  The space after the alias value causes Bash to resolve the first word after the alias value as an alias as well (see [Bash docs](https://www.gnu.org/software/bash/manual/bash.html#Aliases)). This means that if `myalias` is an alias in the current user's environment, then the following invocation succeeds:
>  ```bash
>  sudo myalias
>  ```
>  However, it's Bash doing the replacement before invoking `sudo` and `sudo` actually never sees the `myalias` alias, only the substituted value. As mentioned, the `sudo` environment itself doesn't have any access to shell aliases, shell functions, or shell builtins.


#### `sudo -s`

 - This starts an interactive _non-login_ shell and sources `$HOME/.bashrc`. That means, the environment consists of the `sudo` environment as defined by the settings in the `sudoers` file plus anything (including shell aliases and functions) defined in the `$HOME/.bashrc` file. The interactive shell also has access to shell builtins like a normal shell.
   - Which `.bashrc` file is sourced depends on the value of the `HOME` variable which can be influenced by the `env_keep` setting. For example, setting `env_keep += HOME` sets the `HOME` variable to the invoking user's home directory (instead of the root user's home directory) which consequently causes the invoking user's `.bashrc` file to be sourced.
 - When the interactive shell starts, it stays in the same directory from which `sudo -s` was invoked.

#### `sudo -i`

- This starts an interactive _login_ shell and sources the entire chain of config files starting from `/etc/profile` down to the root user's `.profile` file.
- Unlike the other two options, it does overwrite some of the settings in the `sudoers` file:
  - The `HOME` environment variable is always set to the root user's home directory, no matter whether it's added to `env_keep` in the `sudoers` file.
  - The `env_reset` setting in the `sudoers` file is always enforced to be true, no matter what its actual value is in the `sudoers` file.
- Due to the forced setting of `HOME`, it's always the root user's shell config that is sourced, and since a login shell is started, the sourced config file is `.profile` (corresponding to the `.bash_profile` file of non-root users).
- When the interactive shell starts, the current working directory is changed to the user's home directory.

#### Recommendations

- For simple one-off commands, use `sudo CMD`.
- If a shell alias, shell function, or shell builtin needs to be executed, or if access to environment variables or other settings defined in a `.bashrc` file is needed, start an interactive shell with `sudo -s`.
- There should never really be a requirement to use `sudo -i`.

## References

1. [`man sudo`](https://linux.die.net/man/8/sudo)
1. [`man sudoers`](https://linux.die.net/man/5/sudoers)
