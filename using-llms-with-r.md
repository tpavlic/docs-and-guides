# Using LLMs with R: A Practical Map of the Approaches

A short field guide to the main ways you can put a large language model to work alongside R, ordered roughly from *least integrated with your R session* to *most*. Written for colleagues and students who already know R and want to choose a setup deliberately rather than by accident.

> **Why R needs its own guide.** Much of the real work in R lives in a *stateful, interactive session*: fitted model objects, data frames whose actual structure matters, plots you just generated. A generic "edit a file, run it, read the output" loop throws that away, which is exactly why LLMs tend to hallucinate function signatures or guess at column types when they cannot see your session. The approaches below differ mostly in how well they close that gap.

> [!NOTE]
> This space moves fast. Versions, product names, and defaults change often. Treat everything below as a starting point and check the linked docs. Last reviewed June 2026.

> [!TIP]
> **TL;DR.** If you just want a recommendation: install [Positron](https://positron.posit.co/) and run [Claude Code](https://www.anthropic.com/claude-code) in its integrated terminal, then register the [`btw`](https://posit-dev.github.io/btw/) MCP server so the agent can read your installed-package documentation and the live objects in your R session. Add [`renv`](https://rstudio.github.io/renv/) for reproducibility. Everything below is context for when that default does not fit; see [the recommended stack](#a-recommended-default-stack) for the full picture.

---

- [Quick chooser](#quick-chooser)
- [1. Separate assistant and R session](#1-separate-assistant-and-r-session)
  - [Copy and paste (stand-alone chat)](#copy-and-paste-stand-alone-chat)
  - [Agents that drive your browser or desktop](#agents-that-drive-your-browser-or-desktop)
  - [Where R runs in the web browser: cloud options](#where-r-runs-in-the-web-browser-cloud-options)
- [2. Assistants built into editors and Integrated Development Environments (IDEs)](#2-assistants-built-into-editors-and-integrated-development-environments-ides)
  - [Purpose-built R IDEs: Positron and RStudio](#purpose-built-r-ides-positron-and-rstudio)
  - [General-purpose editors you assemble: VS Code](#general-purpose-editors-you-assemble-vs-code)
  - [Choosing between Positron, RStudio, and VS Code](#choosing-between-positron-rstudio-and-vs-code)
- [3. Agentic CLI: Claude Code, Codex, and Gemini](#3-agentic-cli-claude-code-codex-and-gemini)
- [4. Native R tooling: ellmer, btw, and mcptools](#4-native-r-tooling-ellmer-btw-and-mcptools)
  - [R as the client: ellmer](#r-as-the-client-ellmer)
  - [R as the server: mcptools and btw](#r-as-the-server-mcptools-and-btw)
  - [btw: the connective layer](#btw-the-connective-layer)
  - [Force multiplier for any editor: the bridge across the setups above](#force-multiplier-for-any-editor-the-bridge-across-the-setups-above)
- [Cross-cutting cautions](#cross-cutting-cautions)
- [Reproducibility and sandboxing](#reproducibility-and-sandboxing)
- [A recommended default stack](#a-recommended-default-stack)
- [Selected references](#selected-references)

---

## Quick chooser

| If you want to...                                       | Reach for                                                                                            |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Ask one-off questions, paste a traceback                | [A stand-alone chat](#copy-and-paste-stand-alone-chat)                                               |
| Gather session context to paste into any chat           | [`btw()`](#btw-the-connective-layer)                                                                 |
| Have an external agent operate R or a cloud session     | [A browser- or desktop-driving agent](#agents-that-drive-your-browser-or-desktop)                    |
| Run R without a local install                           | [A cloud notebook or IDE](#where-r-runs-in-the-web-browser-cloud-options)                            |
| Get inline completions and chat without leaving the IDE | [An in-IDE assistant](#2-assistants-built-into-editors-and-integrated-development-environments-ides) |
| Hand off a multi-file task to an agent                  | [An agentic CLI](#3-agentic-cli-claude-code-codex-and-gemini)                                        |
| Let an agent see your real session and package docs     | [R as an MCP server](#r-as-the-server-mcptools-and-btw)                                              |
| Call an LLM *from inside* your R code                   | [`ellmer`](#r-as-the-client-ellmer)                                                                  |
| Pin versions or sandbox an autonomous agent             | [Docker and `renv`](#reproducibility-and-sandboxing)                                                 |

## 1. Separate assistant and R session

In this category the LLM runs as its own application, completely outside your R toolchain, and the two are bridged either by **you** (copy and paste) or by **UI automation** (an agent that operates your screen). It is the least integrated family of approaches, which makes it the easiest to start with and a dependable fallback when nothing fancier is set up. The tradeoff is that the assistant has little or no native understanding of R's internals; it sees text or pixels, not your fitted objects.

### Copy and paste (stand-alone chat)

Pasting code and output into a chat window works and needs no setup. Most people do this in [Claude Desktop](https://claude.ai/download), [claude.ai](https://claude.ai/), [ChatGPT](https://chatgpt.com/), or [Gemini](https://gemini.google.com/). It is fine for quick conceptual questions, explaining an error, or sketching an approach. The cost is friction and context loss: you hand-copy column names, sample rows, error messages, and structure, and the model still guesses at things it cannot see. You can cut most of that friction with the [`btw()`](#btw-the-connective-layer) function, which gathers a structured snapshot of your session and copies it to the clipboard, ready to paste into any chat. But know that the rest of this guide exists to remove the copying entirely.

### Agents that drive your browser or desktop

A newer and more automated variant: instead of you ferrying text, an agent operates an application on your behalf. These are genuinely useful but still maturing, so treat them as experimental and keep a human in the loop.

- **Browser-operating agents** such as [Perplexity's Comet](https://www.perplexity.ai/comet), [Claude for Chrome](https://www.anthropic.com/news/claude-for-chrome), and Chrome's built-in [Gemini "Auto Browse"](https://blog.google/products-and-platforms/products/chrome/gemini-3-auto-browse/) can read and act inside web pages. Pointed at a browser-based R session (see below), they can edit a cell, run it, and read the result without your copy-paste.
- **Desktop-operating agents.** ChatGPT's [Work with Apps on macOS](https://help.openai.com/en/articles/10119604-work-with-apps-on-macos) can read and edit code in supported editors, including VS Code and its forks (which covers Positron), JetBrains IDEs, and Xcode, as well as Terminal, iTerm, and Warp. OpenAI's newer Codex desktop app goes further, controlling local tools and acting on whatever is on screen. For R specifically, the editor and terminal integrations cover R code open in a VS Code fork like Positron, or an `R` session running in a terminal; the more autonomous screen-control agents can in principle drive other setups too, such as RStudio or the standalone R GUI (R.app on macOS). These earn their keep when your R lives somewhere without a built-in assistant, so an RStudio user or someone in a bare terminal gains more here than a Positron user, who already has an assistant in the IDE.

> [!CAUTION]
> An agent that can drive your browser or desktop is a much larger security surface than a chat box. It is exposed to prompt injection from any page or file it reads, and it acts with your privileges. Grant access narrowly, review actions before they run, and never point one at sensitive systems unattended.

### Where R runs in the web browser: cloud options

Browser-driving agents pair naturally with R that already lives in the browser, and cloud R is useful on its own when you want to avoid a local install or give students a uniform environment.

- [Posit Cloud](https://posit.cloud/) gives you a hosted RStudio environment in the browser, with shareable projects, which is handy for teaching and collaboration.
- [Google Colab](https://colab.research.google.com/) supports R two ways: switch the runtime to R under *Runtime > Change runtime type*, or stay in a Python runtime and use the `rpy2` magics (`%load_ext rpy2.ipython`, then `%%R` cells) to mix R and Python. Note that the R runtime resets when you reopen a notebook, so package installs must be rerun, and Drive mounting is limited in the R runtime.

> *Aside:* [WebR](https://docs.r-wasm.org/webr/latest/) is R compiled to WebAssembly, so it runs entirely in the browser with no server. It is excellent for *embedding* R into web pages and interactive teaching materials, but it is a poor host for an agentic coding loop. Use it to ship R-backed widgets, not to write code with an assistant.

## 2. Assistants built into editors and Integrated Development Environments (IDEs)

These live inside your editor and can see your files and editor context. Worth setting expectations up front: most of them reason from your *open files and project*, not from the objects in your *running* R session. An assistant can read a script that fits a model, but it does not, by default, see the fitted object sitting in your environment. Positron Assistant is somewhat more session-aware than plain autocompletion, but the general rule holds, and closing that gap is what [section 4](#4-native-r-tooling-ellmer-btw-and-mcptools) is for.

These tools split into two families. **Purpose-built R IDEs** ship the assistant and the data-science panes together, so there is little to assemble. **General-purpose editors** are the reverse: the assistant ecosystem is the main draw, and the R niceties are something you add back yourself. Switching to a general editor is often the shortest path to the best LLM tooling, which simply relocates the question to *how do I turn a general editor into a good R IDE* — answered in the second subsection.

### Purpose-built R IDEs: Positron and RStudio

- **[Positron](https://positron.posit.co/) Assistant.** Positron is Posit's polyglot, VS Code-based IDE (built on Code-OSS) aimed at R and Python side by side. Its Assistant is built in and Claude-powered, with both a chat sidebar and an agent mode; the agent can scan your project, detect the R version, write code, and run it with your approval. No extra wiring required. It bills against your own Anthropic key, so cost is a real consideration for token-heavy agentic loops. If you are coming from RStudio, Positron also ships interactive Walkthroughs (on the *Welcome* page, or via *Help > Welcome*), including a **"Migrating from RStudio to Positron"** guide that maps familiar RStudio features to their Positron equivalents.
- **[RStudio](https://posit.co/download/rstudio-desktop/) with GitHub Copilot.** Copilot has been built into RStudio natively since the 2023.09 release; you do not install a separate plugin. Enable it under *Tools > Global Options* in the **Copilot** pane (newer builds expose an **Assistant** pane with a "Use code assistant" dropdown instead), let it download the Copilot Agent components, then sign in with the device-code flow and authorize the plugin. It needs a [GitHub Copilot subscription](https://github.com/features/copilot), which is free for verified students and educators. You get ghost-text completions drawn from your active document, and you can widen the context with the "Index project files" setting. Posit has also been rolling its own Assistant into RStudio. RStudio remains the reference experience for the Shiny run/hot-reload loop, the devtools Build pane, and deep R Markdown and Sweave support.

### General-purpose editors you assemble: VS Code

[VS Code](https://code.visualstudio.com/) is the leading example, and the reason to reach for it in R work is that it is the most flexible host for AI extensions: GitHub Copilot's agent mode, [Claude Code](https://www.anthropic.com/claude-code), and OpenAI's Codex all ship as VS Code extensions. The catch is that you start without the data-science panes (console, Data Explorer, Variables, plots) that Positron and RStudio provide out of the box, so making VS Code a genuine RStudio competitor means assembling a few pieces yourself. (Positron, notably, is itself a VS Code fork that Posit has pre-assembled into exactly this, so the steps below are roughly what it does for you.)

1. **Language support.** Install the [R extension](https://marketplace.visualstudio.com/items?itemName=REditorSupport.r) for syntax highlighting, completion, the terminal integration, and the data, plot, and variable viewers. Posit's [Air](https://posit-dev.github.io/air/) (a fast R formatter and language server) installs as its own extension, found by searching "Air" in the Extensions pane; it bundles its own binary, so there is nothing else to set up, and it comes pre-installed in Positron.

2. **Plots.** Once the R session is attached (see the troubleshooting tip below), plots render automatically in a tab beside your editor through the extension's built-in PNG viewer — no extra setup, since plot capture is part of the session watcher. For a more interactive viewer that resizes, zooms, keeps a plot history, and follows your light/dark theme like RStudio's pane, there are two solid options:

   - [`httpgd`](https://nx10.dev/httpgd/articles/b01_vscode.html) upgrades the extension's built-in viewer in place. Install it in R with `install.packages("httpgd")`, then enable the `r.plot.useHttpgd` setting (Cmd/Ctrl+`,` to open Settings, then search for it). **Restart the R session afterward:** the graphics device is chosen when the session attaches, so the toggle does nothing until you start a fresh R terminal. Plots then open as a resizable SVG tab beside your editor.

   - For the closest thing to RStudio's Plots pane, the third-party [R Plot Pro](https://marketplace.visualstudio.com/items?itemName=ofurkancoban.r-plot-pro) extension lives in the bottom panel beside the terminal and wraps plots in familiar chrome: a toolbar for paging back and forth through plot history, zooming, and saving or exporting, plus a thumbnail history strip down the side. RStudio migrants tend to find it the most recognizable of the options.

3. **AI layer.** This is the payoff that motivated the move to VS Code in the first place: layer on whichever assistant you want, each as an extension — GitHub Copilot (now with an agent mode), Claude Code, or Codex. Google is the exception: its Gemini Code Assist *extension* and Gemini CLI stop serving individuals [on June 18, 2026](https://developers.google.com/gemini-code-assist/resources/release-notes), and the successor, [Antigravity](https://antigravity.google/), is not another extension but a separate agentic IDE (a Cursor-style VS Code fork; see [choosing an editor](#choosing-between-positron-rstudio-and-vs-code)) — so the bolt-on path for Gemini in VS Code is effectively ending.

> [!TIP]
> If a fresh R terminal in VS Code reports **"R: (not attached)"** and you see `could not find function '.vsc.attach'`, the extension's session watcher failed to initialize (reported with recent R, such as 4.6.0). A workaround that circulates among users is to source the extension's init script from your `.Rprofile` — open it with `usethis::edit_r_profile()`, which finds the right file on Windows, macOS, and Linux alike, then add:
>
> ```r
> if (interactive() && Sys.getenv("RSTUDIO") == "") {
>   init_path <- file.path(
>     Sys.getenv(if (.Platform$OS.type == "windows") "USERPROFILE" else "HOME"),
>     ".vscode-R", "init.R"
>   )
>   source(init_path)
>   .First.sys()  # workaround for the missing .vsc.attach()
> }
> ```
>
> See the [vscode-R issue thread](https://github.com/REditorSupport/vscode-R/issues/1696#issuecomment-4343488948) for context and discussion.

### Choosing between Positron, RStudio, and VS Code

Positron and RStudio are both from Posit and install side by side; Posit maintains both and has set no deprecation timeline for RStudio. Positron is under much heavier active development and wins on polyglot R/Python work, multiple R versions and concurrent sessions, the Data Explorer, the VS Code extension marketplace, remote sessions, and running the interpreter separately from the IDE (so a crash in R need not take the IDE down); it also opens Jupyter notebooks natively with full R and Python kernels, which RStudio does not do at all. RStudio still leads on Shiny and package development. Posit's own [feature comparison](https://positron.posit.co/migrate-rstudio-compare.html) weighs the two in detail.

Plain VS Code sits apart from both: it is the most flexible host for AI extensions but the least specialized for R out of the box, since the data-science panes are something you [assemble yourself](#general-purpose-editors-you-assemble-vs-code). It makes the most sense when R is one language among several in an editor you already live in, or when habit or institutional policy fixes your editor and you would rather extend it than adopt a dedicated IDE. For an R-first user, though, Positron is itself a pre-assembled VS Code fork, so it delivers most of what you would otherwise build by hand with less friction — which is why it, and not plain VS Code, is the default recommendation here. A common pattern among experienced users is to keep more than one installed and open whichever fits the task.

> [!NOTE]
> **The agentic VS Code forks.** Cursor, Windsurf, and Google's [Antigravity](https://antigravity.google/) (the successor to Gemini Code Assist) are VS Code forks tuned for agent-first general development rather than R. They can usually install the R extension from the OpenVSX registry these forks use, but that support is less proven, which makes them the least R-specialized editors covered here. Antigravity's free tier is appealing for students but has tightened since launch, so check current limits before relying on it.

> [!IMPORTANT]
> Integrating agentic AI into an editor is not the same as giving that AI your R session. In Positron, RStudio, and VS Code alike — and in the agentic forks above — the built-in assistants and agent modes work primarily from your files and editor context; by default they do not see the data frames and fitted objects in your *running* session, and instead infer structure from your code and guess the rest. To let any of these tools read your real session state (and the documentation of the packages you actually have installed), wire up the [native bridge in section 4](#4-native-r-tooling-ellmer-btw-and-mcptools) as well. The IDE and the bridge are complementary, not alternatives.

## 3. Agentic CLI: Claude Code, Codex, and Gemini

An agentic command-line tool reads and edits files across your project, runs commands (`Rscript`, `R CMD check`, tests), reads the output, and iterates. This is the strongest option for multi-file refactors, package work, and "do this whole task" handoffs. Three terminal agents dominate in 2026, all of which support MCP (see [section 4](#4-native-r-tooling-ellmer-btw-and-mcptools)), subagents, and whole-codebase awareness:

- **[Claude Code](https://www.anthropic.com/claude-code)** (Anthropic). The strongest at multi-file reasoning and code quality, with skills and multi-agent orchestration. Installs via `npm`. Premium, through a Claude subscription or the API. Sessions can be handed off to and continued from the Claude desktop and mobile apps.
- **[Codex CLI](https://developers.openai.com/codex/cli/)** (OpenAI). Open-source (Apache 2.0), rebuilt in Rust for speed, notably token-efficient, with strong kernel-level sandboxing. Bundled with ChatGPT plans, and paired with a macOS desktop app and an IDE extension, so you can start in the terminal and continue in the app.
- **[Gemini CLI](https://github.com/google-gemini/gemini-cli)** (Google). Open-source, with the largest context window (around 1M tokens) and historically a generous free tier, plus a read-only "Plan Mode" that proposes a strategy before touching files. Long the natural pick for very large codebases or budget-conscious work — but note that Google is folding Gemini CLI for individuals into its new [Antigravity](https://antigravity.google/) CLI [as of June 18, 2026](https://developers.google.com/gemini-code-assist/resources/release-notes), and Antigravity's free terms have already been tightened since launch, so verify current limits before relying on them.

Most people end up using more than one for different tasks. The differences in raw code quality matter less than fit with your existing accounts, your budget, and how much autonomy you are comfortable granting.

**Running them inside an IDE terminal.** You do not have to choose between a CLI agent and an IDE. Run the agent in your IDE's integrated terminal and you get both: the agent's full feature set plus the editor's R session, plots pane, Data Explorer, and debugger right beside it. Positron is a particularly good host because it runs these tools in *terminal mode* rather than as a constrained editor extension, so you lose none of the CLI's capabilities.

**The statefulness catch.** A CLI agent runs as a separate process *alongside* R, not inside it. Left to its own devices it edits scripts and runs them with `Rscript`, which spins up a fresh session each time, so it does not automatically see the objects in your interactive session, any more than the in-IDE assistants in [section 2](#2-assistants-built-into-editors-and-integrated-development-environments-ides) do. Hosting the agent inside an IDE does not change this: the R extension's session viewers, plots pane, and Data Explorer serve *you*, not the agent, so the editor's live session stays invisible to a CLI tool running in its terminal. That is fine for a lot of file-level work, but it is the single biggest reason generated R code misreads your data or a fitted object. The fix is to give the agent a way to query your live session, which is what the next section provides.

## 4. Native R tooling: ellmer, btw, and mcptools

This is Posit's stack for wiring R and LLMs together directly. It runs in two directions, with one package acting as the connective tissue between them:

- **R as the client:** call an LLM from inside your R code ([`ellmer`](https://ellmer.tidyverse.org/)).
- **R as the server:** expose your R session and tools to an outside agent like Claude Code ([`mcptools`](https://posit-dev.github.io/mcptools/), via [`btw`](https://posit-dev.github.io/btw/)).
- **The connective layer:** [`btw`](https://posit-dev.github.io/btw/) gathers R context and feeds it to either direction.

### R as the client: ellmer

Use this when you do not want an assistant so much as to *program with* a model: classify free text, extract structured fields, build a domain chatbot, or run a model over many rows. [`ellmer`](https://ellmer.tidyverse.org/) is the core package. It supports many providers (Anthropic, OpenAI, Google, AWS Bedrock, Azure, and more), with streaming, tool/function calling, and structured data extraction. Chat objects are stateful R6 objects, so each turn builds on the last.

```r
library(ellmer)

chat <- chat_anthropic(
  system_prompt = "You are a terse assistant. American English.",
  model = "claude-sonnet-4-6"
)

# interactive REPL in the console (or live_browser(chat) for a UI)
live_console(chat)

# structured extraction returns typed data, not prose
chat$chat_structured(
  "Heat-shocked at 42C for 30 min, then 25C recovery.",
  type = type_object(
    stressor   = type_string(),
    temp_c     = type_number(),
    duration_m = type_number()
  )
)
```

Watch your token usage with `token_usage()`; cost grows with conversation length because each turn resends the full context. Around `ellmer` sit a few companions: [`ragnar`](https://ragnar.tidyverse.org/) for Retrieval-Augmented Generation over your own documents, [`shinychat`](https://posit-dev.github.io/shinychat/) for wrapping a chat in a Shiny UI to share with a lab or class, and [`vitals`](https://vitals.tidyverse.org/) for evaluating model or prompt quality when you are benchmarking rather than just using.

### R as the server: mcptools and btw

Here R is the thing being called. The [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) lets a coding agent call tools you expose, and the payoff for R specifically is twofold: the agent can **query the live state of your R session** (the data frames and fitted objects in your global environment) and **read the documentation of the packages you actually have installed**. That directly addresses the statefulness limitation from [section 3](#3-agentic-cli-claude-code-codex-and-gemini), and to a lesser extent the file-only context of the in-IDE assistants in [section 2](#2-assistants-built-into-editors-and-integrated-development-environments-ides); instead of guessing function arguments or re-deriving your data, the agent inspects the real thing.

[`mcptools`](https://posit-dev.github.io/mcptools/) is the protocol layer. In practice you start from [`btw`](https://posit-dev.github.io/btw/), which ships a ready-made server exposing those documentation and session tools. Register it with Claude Code from a terminal:

```bash
claude mcp add -s user r-btw -- Rscript -e "btw::btw_mcp_server()"
```

To let the agent reach into your *running* interactive sessions rather than only spawning fresh ones, add a session hook to your `.Rprofile`:

```r
# usethis::edit_r_profile() to open it
if (interactive()) {
  btw::btw_mcp_session()
}
```

Drop to `mcptools` directly when you want to expose *arbitrary custom R functions* as tools, or to use R as an MCP *client* that connects out to other MCP servers from an `ellmer` chat.

> [!NOTE]
> `mcptools` was formerly `acquaint`. If you see `acquaint::mcp_server()` in older material, the modern equivalent is `btw::btw_mcp_server()`.

> [!WARNING]
> Session access means the agent can read your global environment. That is the point, but be deliberate about it if a session holds sensitive or very large objects.

### btw: the connective layer

[`btw`](https://posit-dev.github.io/btw/) is best understood as the glue across everything above; its own tagline is that it "supercharges `ellmer`." It offers three things from one package:

1. **Context to clipboard.** `btw()` describes objects, package or function docs, and files, and copies the result to your clipboard for any external chat. This is what makes the [copy-and-paste workflow](#copy-and-paste-stand-alone-chat) far less painful.

   ```r
   library(btw)
   btw(mtcars)                                      # describe a data frame
   btw("{dplyr}", "How do I group and summarize?")  # add package docs plus a question
   ```

2. **An in-IDE chat with session access.** `btw_client()` returns an `ellmer` chat preloaded with `btw` tools, and `btw_app()` launches a Shiny chat interface that runs in your IDE *with access to your R environment*. Defaults (provider, model, tools, instructions) can be pinned per project in a `btw.md` file, and it also reads `AGENTS.md` or `CLAUDE.md` project context.

   ```r
   chat <- btw_client()   # an ellmer chat that can see your session
   chat$chat("Why is my lmer model failing to converge?")
   ```

3. **Tools for other clients and servers.** `btw_tools()` returns tool objects you can register with any `ellmer` chat, and `btw_mcp_server()` is the MCP server used in the section above. Same tools, whether R is the client or the server.

> [!CAUTION]
> `btw`'s optional run-R tool executes model-written code in your global environment with no sandbox and, as of current `shinychat` and `ellmer`, no review-before-execution step. It is off by default for good reason. Enable it only in a throwaway or sandboxed session, never in a publicly reachable app.

### Force multiplier for any editor: the bridge across the setups above

The native tooling is not a fourth approach you adopt *instead* of the others — it is what upgrades whichever approach from sections 1–3 you already use. The failure named at the top of this guide, where the model guesses at column types or invents a function signature, is exactly what the bridge removes: it hands the model your real session state and the documentation of the packages you actually have installed. How much that helps scales with how tightly it is wired in.

| Your setup (exemplar)                                       | What the `btw`/`mcptools` bridge adds                                                                                           |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Stand-alone chat ([claude.ai](https://claude.ai/), ChatGPT) | `btw()` snapshots your objects and docs to the clipboard — a manual paste with no live link, but far better than hand-copying   |
| In-IDE assistant (Positron Assistant; VS Code + Copilot)    | register `btw_mcp_server()` and the chat can pull live objects and installed-package docs on demand, not just your open files   |
| Agentic CLI (Claude Code in a Positron or VS Code terminal) | the MCP server **plus** the `btw_mcp_session()` hook lets the agent inspect your *running* session mid-task, then edit to match |

**A worked example.** Say Claude Code is running in Positron's terminal and you ask it to fix a `dplyr` pipeline that errors on a grouping column. Without the bridge, the agent sees only the script: it guesses the column is `group` (it is actually `treatment_grp`), edits the file, runs it with `Rscript`, reads the error, and guesses again — a slow round-trip that fails outright if the data lives only in your interactive session and never as a file the agent can source. With `btw_mcp_session()` active, it instead calls the session tool, reads the data frame's real column names and types, confirms the grouping variable, and makes the correct edit on the first pass. The same holds for a non-converging model: it can read the fitted object's structure and the actual `lmer` call rather than inferring slot names.

The wiring is just the two pieces shown earlier in this section — register `btw_mcp_server()` with Claude Code, and add `btw_mcp_session()` to your `.Rprofile` so the agent reaches the session you are actually working in. That is what turns the [statefulness catch](#3-agentic-cli-claude-code-codex-and-gemini) into a non-issue, and why the [recommended stack](#a-recommended-default-stack) pairs Claude Code with the `btw` MCP server rather than running it blind.

## Cross-cutting cautions

- **Verify generated statistics code.** Plausible-looking R is not correct R. Model-selection logic, contrast coding, random-effects structure, and the meaning of a fitted object's slots all reward a careful read. Treat agent output as a fast first draft to react to, not an answer.
- **Mind what the assistant can see, and do.** Live-session access is data exposure; UI-driving and desktop agents go further and act with your privileges, which widens the prompt-injection surface. Decide consciously what each tool can reach before you wire it up.
- **Cost and keys.** Agentic and in-IDE-assistant workflows bill against your API key and scale with context length and turns. For students and self-funded work, this matters; prefer cheaper models for routine tasks and reserve top-tier models for hard problems.
- **Reproducibility.** If an LLM touched a result that goes into a paper, the pipeline should still run without it. Pin versions, script the calls, and keep a human-readable record of what the model did.

## Reproducibility and sandboxing

This is less an integration approach than the infrastructure that makes the others safe to lean on. Agents are most useful when their environment is pinned and their autonomy is bounded.

- **[`renv`](https://rstudio.github.io/renv/)** snapshots your package versions into a lockfile so a generated analysis runs the same way next month and on a student's machine.
- **Docker** (for example the [rocker](https://rocker-project.org/) images) gives a clean, reproducible R environment and a sandbox in which an agentic CLI can run with less risk to your real filesystem. The tradeoff is that container isolation complicates the live-session introspection of [section 4](#r-as-the-server-mcptools-and-btw), so reach for Docker when reproducibility or isolation is the priority rather than as your default interactive loop. A common pattern: develop interactively with the `btw` bridge, then verify the final result inside a clean container.

## A recommended default stack

1. **[Positron](https://positron.posit.co/)** as the IDE (keep RStudio around for Shiny and package Build-pane work).
2. **[Claude Code](https://www.anthropic.com/claude-code) in Positron's terminal** for agentic, multi-file tasks.
3. **[`btw`](https://posit-dev.github.io/btw/) MCP server** registered, with `btw_mcp_session()` in `.Rprofile`, so the agent sees your package docs and live objects.
4. **[`ellmer`](https://ellmer.tidyverse.org/)** (plus [`ragnar`](https://ragnar.tidyverse.org/) or [`shinychat`](https://posit-dev.github.io/shinychat/) as needed) when the model belongs *inside* your R code, with `btw()` on hand for quick context whenever you drop back to a stand-alone chat.
5. **[`renv`](https://rstudio.github.io/renv/)** always; **[Docker](https://rocker-project.org/)** when you need a clean room or a sandbox.

## Selected references

- ellmer: <https://ellmer.tidyverse.org/>
- btw: <https://posit-dev.github.io/btw/>
- mcptools: <https://posit-dev.github.io/mcptools/>
- ragnar: <https://ragnar.tidyverse.org/>
- Positron: <https://positron.posit.co/>
- Positron vs RStudio feature comparison: <https://positron.posit.co/migrate-rstudio-compare.html>
- R extension for VS Code: <https://marketplace.visualstudio.com/items?itemName=REditorSupport.r>
- httpgd plot viewer in VS Code: <https://nx10.dev/httpgd/articles/b01_vscode.html>
- R Plot Pro extension: <https://marketplace.visualstudio.com/items?itemName=ofurkancoban.r-plot-pro>
- RStudio GitHub Copilot guide: <https://docs.posit.co/ide/user/ide/guide/tools/copilot.html>
- Posit Cloud: <https://posit.cloud/>
- Google Colab: <https://colab.research.google.com/>
- Claude Code: <https://www.anthropic.com/claude-code>
- Codex CLI: <https://developers.openai.com/codex/cli/>
- Gemini CLI: <https://github.com/google-gemini/gemini-cli>
- Google Antigravity (successor to Gemini Code Assist / Gemini CLI for individuals): <https://antigravity.google/>
- ChatGPT "Work with Apps" on macOS: <https://help.openai.com/en/articles/10119604-work-with-apps-on-macos>
- rocker (Docker images for R): <https://rocker-project.org/>
- WebR: <https://docs.r-wasm.org/webr/latest/>

---

*MIT License. Copyright © 2026 Theodore P. Pavlic. Shared as a starting point; verify against current documentation, since this tooling changes quickly.*
