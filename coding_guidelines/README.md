# General coding guidelines

Basically, you should always adhere to a *language's specific coding practices*:

* [Concourse](concourse.md)
* [Docker](docker.md)
* [Helm](helm.md)
* [Terragrunt / OpenTofu](terragrunt.md)

We also specify some [guidelines around using Git](git.md)

However, we also define a set of standard practices, described below.

Although we use different environments to do our work, everything we do is rooted in Linux/POSIX based environments. Therefore we should adhere to these standards for our text based files.

> [!TIP]
> Most of these formatting options can be automated via your editor and/or linter, so you should install the language's specific package in your editor.

* Use Unix-style newlines (`\n`) only
* [A file should have an extra newline (`\n`) at the end](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap03.html#tag_03_206)

> [!NOTE]
> By default vi(m) does not show this line but handles it correctly.

* Prefer spaces (2 or 4) above tabs, unless the used language has another standard
* Use descriptive variable/function/... naming, instead of very short unmeaningful ones
* Prefer underscore based variable/function/... naming, `a_name_for_something`, unless the used language has another standard
* Properly align and/or format your code, unless the used language has another standard
* Remove unnecessary whitespace, unless it serves functionality and/or better readability

## Example editor configurations

### (n)vim

```vimrc
set colorcolumn=80                         " Mark the 80th column
set expandtab                              " Use spaces instead of tabs
set fixeol                                 " Add blank line at the end of a file if missing
set list                                   " Show invisible characters
set listchars=tab:▸\ ,trail:·,eol:¬,nbsp:_ " Set invisible characters
set modeline                               " Respect modeline in files
set shiftwidth=2                           " Make indentation (>) as wide as two spaces
set tabstop=2                              " Make tabs as wide as two spaces
```

### VSCode

```json
{
  "editor.detectIndentation": true,
  "editor.formatOnPaste": true,
  "editor.formatOnSave": true,
  "editor.renderWhitespace": "boundary",
  "editor.rulers": [80],
  "editor.tabSize": 2,
  "editor.wordWrap": "on",
  "files.eol": "\n",
  "files.insertFinalNewline": true,
  "files.trimFinalNewlines": true,
  "markdown.extension.tableFormatter.enabled": false,
}
```

#### Recommended extensions

* `davidanson.vscode-markdownlint`
* `ms-azuretools.vscode-docker`
* `ms-kubernetes-tools.vscode-kubernetes-tools`
* `ms-vscode-remote.remote-wsl` (when using WSL)
* `pivotal.vscode-concourse`
* `redhat.vscode-yaml`
* `yzhang.markdown-all-in-one`
* `signageos.signageos-vscode-sops`
* `skyscrapers-engineering.vscode-kms`
* `gamunu.opentofu`

### Atom

```json
"*":
  editor:
    preferredLineLength: 120
    showIndentGuide: true
    showInvisibles: true
  "line-ending-selector":
    defaultLineEnding: "LF"
  "tree-view":
    hideVcsIgnoredFiles: false
  whitespace:
    removeTrailingWhitespace: true
```
