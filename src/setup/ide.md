# IDE

In addition to the Rust toolchain itself, you will obviously need a code editor or an IDE with Rust support. Popular options include:

* [Rust Rover](https://www.jetbrains.com/rust/) — a very powerful IDE from JetBrains designed specifically for Rust. Free for open-source development and learning, but a license is required for commercial use.
* [VSCode](https://code.visualstudio.com/) — a powerful, free, and very flexible open-source development environment.
* [Zed](https://zed.dev/) — a new, free, and very fast development environment written in Rust.
* [NeoVim](https://neovim.io/) — a powerful terminal-based editor with a steep learning curve, but great flexibility and extensibility. NeoVim is a fork of the original Vim, where Lua is used for writing plugins instead of VimScript.
* [Helix](https://helix-editor.com/) — a "Vim-like" editor written in Rust. Not as flexible as NeoVim, but more user-friendly and simpler.

If you already use one of the JetBrains IDEs and are satisfied with it, as well as with its license and cost, you can simply install RustRover and move on to the next chapter.

## rust-analyzer

Among all the IDEs listed above, only RustRover has its own implementation of a Rust parser and analyzer. All the others work with Rust code via [rust-analyzer](https://rust-analyzer.github.io/).

rust-analyzer is an application that analyzes Rust code and provides features such as:

* type inference
* code completion
* go to variable/function/type definition
* find all usages of a function/type
* and more

Code editors interact with rust-analyzer via LSP (Language Server Protocol) — a protocol specifically designed for communication between editors and code analyzers.

![](img/rust_analyzer.svg)

This means that before any of the editors listed above can work with Rust, you need to install rust-analyzer. You can do this with the following command:

```
rustup component add rust-analyzer
```

After that, you can proceed with installing and configuring your IDE.

## VSCode

If you have not worked with JetBrains IDEs before, or have worked with them but found them unsuitable, the first recommended option to try is VSCode.

VSCode is a code editor that, out of the box, does little more than editing text, but has a large ecosystem of plugins that can easily turn it into a powerful IDE.

First, download VSCode from the official website: [https://code.visualstudio.com/download](https://code.visualstudio.com/download). The website offers the following download options:

* an installer for Windows
* packages for Linux
* a standalone archive

Choose whichever option is most convenient for you.

After downloading, launch VSCode and open the extensions view by clicking the icon shown in the image below:

![](img/vscode-init_screen.png)

In the context menu that appears, select "Extensions", after which the following panel should open:

![](img/vscode-install_extension.png)

In the upper-left corner, there is a search field for extensions. Type "rust-analyzer" into it, and once the extension appears, click "Install".

Generally, this is already enough to start working with Rust. However, it is also recommended to install the following extensions:

* Even Better TOML — syntax highlighting for TOML files\
  ([plugin page link](https://marketplace.visualstudio.com/items?itemName=tamasfe.even-better-toml))
* vscode-icons — more intuitive icons in the file tree\
  ([plugin page link](https://marketplace.visualstudio.com/items?itemName=vscode-icons-team.vscode-icons))\
  After installation, select in the menu:\
  File -> Preferences -> Theme -> File Icon Theme -> VSCode Icons

***

Now let’s test that everything works.

Create a Rust project: open a terminal, navigate to a convenient directory (using the `cd` command), and run:

```
cargo new test_rust_project
```

This command will create a new directory called `test_rust_project` containing a Rust project.

In VSCode, select File -> Open Folder and choose the`test_rust_project` directory.

Your newly created project should now open. In the file tree on the left, click on `src/main.rs`, and the file contents should appear in the code editor on the right.

Above the `main` function (`fn main`), you should see an arrow with the label Run.

![](img/vscode-run.png)

Click Run. If a terminal appears at the bottom of the window and displays the line "Hello, World!", then everything is configured correctly.

## Zed

Zed is a young IDE (its first beta release was in 2023), written in Rust and WGPU (a library for cross-platform GPU graphics).

Zed’s main focus is performance and efficient use of hardware resources. If you are sensitive to latency in GUI applications, you should definitely try Zed.

You can download Zed from the official website: [https://zed.dev/download](https://zed.dev/download)

After launching, Zed looks like this (of course, a dark theme is also available):

![](img/zed_start.png)

Zed works with rust-analyzer out of the box, so no additional extensions are required.

Create a test Rust project: open a terminal, navigate to a convenient directory (using the cd command), and run the following command:

```
cargo new test_rust_project
```

Now open the newly created project folder in Zed. You can do this either by selecting "Open Project" in the central panel, or by clicking the ≡ icon in the top-left corner and choosing File -> Open Folder.
![](img/zed_run.png)

In the project file tree on the left, select src/main.rs. The file contents should appear in the editor pane on the right.

To the left of the main function, you should see a run icon — ▶. Click it, and the program output "Hello, World!" should appear in the console at the bottom.

## NeoVim

If you are a NeoVim user, you already know everything 😉

If you want to try Vim but are intimidated by the configuration process, consider trying [Helix](https://docs.helix-editor.com/) instead.
