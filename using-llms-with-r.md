# Using LLMs with R: The editor, the assistant, and your live session

A short field guide to using large language models (LLMs) alongside R. It begins with the three parts of any R and LLM setup: the editor, the assistant, and the connection to your live R session. It then works through the options for each, including running an MCP server inside R so an assistant can read your session and your installed-package documentation. This guide is written for those who already know R.

> [!NOTE]
> This space moves fast. Versions, product names, and defaults change often. Treat everything below as a starting point, and check the linked docs. Last reviewed August 2026.

---

- [1. Overview: The editor, the assistant, and accessing the R session](#1-overview-the-editor-the-assistant-and-accessing-the-r-session)
- [2. Choosing an editor](#2-choosing-an-editor)
  - [Purpose-built R IDEs: RStudio and Positron](#purpose-built-r-ides-rstudio-and-positron)
  - [General-purpose editors you assemble: VS Code](#general-purpose-editors-you-assemble-vs-code)
  - [Choosing between RStudio, Positron, and VS Code](#choosing-between-rstudio-positron-and-vs-code)
  - [Where R runs in the web browser: cloud options](#where-r-runs-in-the-web-browser-cloud-options)
- [3. Choosing an assistant: model and harness](#3-choosing-an-assistant-model-and-harness)
  - [Posit Assistant](#posit-assistant)
  - [Vendor agents: Claude Code and Codex](#vendor-agents-claude-code-and-codex)
  - [Bring your own key (BYOK)](#bring-your-own-key-byok)
  - [Copy and paste (stand-alone chat)](#copy-and-paste-stand-alone-chat)
  - [Agents that drive your browser or desktop](#agents-that-drive-your-browser-or-desktop)
- [4. Connecting an assistant to your live R session](#4-connecting-an-assistant-to-your-live-r-session)
  - [The statefulness catch](#the-statefulness-catch)
  - [R as an MCP server: mcptools and btw](#r-as-an-mcp-server-mcptools-and-btw)
  - [btw: the connective layer](#btw-the-connective-layer)
  - [What the bridge adds to each assistant](#what-the-bridge-adds-to-each-assistant)
- [5. Using an LLM inside R code: ellmer](#5-using-an-llm-inside-r-code-ellmer)
- [Cross-cutting cautions](#cross-cutting-cautions)
- [Use-case quick reference](#use-case-quick-reference)
- [Selected references](#selected-references)

---

## 1. Overview: The editor, the assistant, and accessing the R session

Using AI coding assistants with R means:

- **Choosing an editor.** This guide focuses on three popular integrated development environments (IDEs) that include both R-specific features and the ability to directly integrate with modern AI tools:

  - **[RStudio](#purpose-built-r-ides-rstudio-and-positron)** – Posit's long-standing R IDE and still the reference experience for Shiny and package development.
  - **[Positron](#purpose-built-r-ides-rstudio-and-positron)** – Posit's newer IDE for R and Python together, with notebooks, remote sessions, and the VS Code extension marketplace.
  - **[VS Code](#general-purpose-editors-you-assemble-vs-code)** – a general-purpose editor, with the R panes something you assemble yourself.

  However, the tips in this guide can also be applied to many other general-purpose editors, including basic text editors without specific hooks for R or AI.

- **Choosing an AI model and harness (AI assistant).** AI models by themselves cannot directly interact with programming files or R or even the programmer themself. Instead, to really be an *AI assistant*, AI models must be paired with a *harness* that gathers input from a human developer and output from an AI model and can possibly read/write/execute resources on the programmer's machine. The choice of model and harness not only shapes the capabilties of the assistant but also the **cost** of using the AI for the developer. For working with R, three methods for getting access to an AI model (as well as their own harness options) include:

  - **A [Posit AI](https://docs.posit.co/posit-ai/) subscription** – Posit's managed cloud-based service for data science applications **in both R and Python**.
    - This cloud-hosted AI model is accessed with the **[Posit Assistant](https://assistant.posit.co/)** harness, which **can directly access live R (and Python) session variables** without additional configuration steps.
      - Posit Assistant is built into RStudio and Positron.
      - Posit Assistant also has a stand-alone terminal user interface (TUI) called `pa`.
      - Posit Assistant supports other models besides Posit AI that are supplied by other vendors.

  - **A [Claude](https://claude.ai/) or [ChatGPT](https://chatgpt.com/) subscription** – General-purpose, vendor-supplied LLMs and agents optimized for multi-file work.
    - Full-featured harnesses are available as extensions for both Positron and VS Code.
    - Separate desktop- and terminal-based harnesses (e.g., Claude Code and Codex) can be used as well.
    - Web-based interfaces are also available but are limited in their integration with a development environment (e.g., can access code on [GitHub](https://github.com) but not locally without uploading files or copying and pasting).
    - Cannot access live R session variables without use of a Model Context Protocol (MCP) server.

  - **An API key (Bring Your Own Key, BYOK)** – Allows for connecting to generic LLM providers either locally or cloud hosted.
    - Most likely path for **free access to sophisticated AI assistance** by way of locally hosted, open-weight models, university- or company-hosted open-weight models, or highly subsidized enterprise gateways to vendor models.
    - Supported natively by Posit Assistant (RStudio, Positron) and VS Code Chat.
      - When used with Posit Assistant, BYOK LLM models have direct access to the R session, but they may not take advantage of it as effectively as Posit AI does.
    - Supported by [OpenCode](https://opencode.ai/), a popular terminal-based open-source AI harness.
    - To gain access to live R session variables, can be used with Posit Assistant or, with most other harnesses, a Model Context Protocol (MCP) server for R (described below).

- **Choosing how to integrate with a live R session.** AI assistants are consistently used alongside a running instance of R so that edited code can be immediately executed, results reviewed, and new changes to code can be made. Locally hosted AI assistants typically have access to local files, but how they directly interact with and manipulate your local R environment can vary. In particular:

  - **Posit Assistant has full R access.** – Posit Assistant in RStudio and Positron (and `pa` in the terminal) automatically have full access to R session variables. However, although the subscription-based Posit AI model has been developed to leverage this feature in Posit Assistant, other BYOK AI models may not use the Posit Assistant capabilities as effectively (if at all).
  - **An [MCP](#4-connecting-an-assistant-to-your-live-r-session) server in R exports access to AI models.** – The Model Context Protocol (MCP) provides a method for connecting AI harnesses to data sources, like R session variables as well as installed-package documentation (which can help reduce hallucinations in code). [Section 5](#4-connecting-an-assistant-to-your-live-r-session) covers how to install an MCP server in R and register it with an AI assistant's harness. This approach applies to the vendor-supplied harnesses for Claude and ChatGPT (and others), the Chat harness built into VS Code, the open-source OpenCode harness, and many others. Many AI models have been trained to leverage tools provided by an MCP server.
  - **AI assistance can be used without a connection to R.** Locally hosted AI harnesses can directly access R files, and code as well as snapshots of the current R session state can be pasted into both locally hosted AI models as well as web-based ones.

To support developer flexibility, the rest of this guide will go into detail about each of these options. The best combination of the above options depends on developer preference and budget. Furthermore, the right mix of the options above can change over time and even across different projects.

> [!NOTE]
> This guide covers using AI models to assist in using R. If instead an AI model is to be used *inside* R code (rather than beside it), the package **[`ellmer`](https://ellmer.tidyverse.org/)** can be used to make R an AI-calling agent/harness itself. See [Section 5](#5-using-an-llm-inside-r-code-ellmer) for details.

> [!IMPORTANT]
> Regardless of the AI option you choose with R, best practice for reproducibility in R recommends using **[`renv`](https://rstudio.github.io/renv/)** to pin package versions and using **[Docker](https://rocker-project.org/)** to create clean, isolated, exportable R environments. Containerization is worth investigating for portability and reproducibility, but it does complicate both calling an AI model from inside R code and using an assistant to help write that code.

## 2. Choosing an editor

Your editor decides which assistants are even available to you, and so it is the first of the three choices even though it is the least about AI. The tools split into two families. **Purpose-built R IDEs** ship the assistant and the data-science panes together, and so there is little to assemble. **General-purpose editors** are the reverse: the assistant ecosystem is the main draw, and the R niceties are something you add back yourself.

### Purpose-built R IDEs: RStudio and Positron

Both come from Posit, both install side by side, and both now host the same AI assistant; the choice between them is about the editor rather than about the AI.

- **[RStudio](https://posit.co/download/rstudio-desktop/)** is the long-standing R IDE and still the reference experience for the Shiny run/hot-reload loop, the devtools Build pane, and deep R Markdown and Sweave support. Its AI layer is **[Posit Assistant](#posit-assistant)**, a downloadable add-in from RStudio 2026.04.0 onward. Older RStudio releases integrated **GitHub Copilot** instead, enabled under *Tools > Global Options*, which gives ghost-text completions drawn from your active document and needs a [Copilot subscription](https://github.com/features/copilot) that is free for verified students and educators.

- **[Positron](https://positron.posit.co/)** is Posit's newer IDE, built for working in R and Python side by side. Posit Assistant ships built in and, from Positron 2026.07, is the default AI experience, replacing the earlier Positron Assistant. Being a VS Code fork, Positron also has the extension marketplace, and so the [vendor agents](#vendor-agents-claude-code-and-codex) below install as extensions in the same window. If you are coming from RStudio, Positron ships interactive Walkthroughs (on the *Welcome* page, or via *Help > Welcome*), including a **"Migrating from RStudio to Positron"** guide that maps familiar RStudio features to their Positron equivalents.

  - To find extensions for various coding assistants (Claude Code, Codex, etc.), use the Extensions pane built into Positron.
  - Positron's sidebar also carries VS Code's own **Chat**, whose icon and panel look much like Posit Assistant's but which has none of the R session tooling. [Section 3](#posit-assistant) covers telling them apart.

### General-purpose editors you assemble: VS Code

[VS Code](https://code.visualstudio.com/) is the most flexible host for AI extensions: GitHub Copilot's agent mode, [Claude Code](https://www.anthropic.com/claude-code), and [OpenAI's Codex](https://chatgpt.com/codex/) all ship as VS Code extensions. The catch is that you start without the data-science panes (console, Data Explorer, Variables, plots) that Positron and RStudio provide out of the box, and so making VS Code a genuine RStudio competitor means assembling a few pieces yourself. Positron is itself a VS Code fork that Posit has pre-assembled into exactly this, and so the steps below are roughly what it does for you.

1. **Language support and session tools.** Install the [R extension](https://marketplace.visualstudio.com/items?itemName=REditorSupport.r). Beyond syntax highlighting, completion, and terminal integration, its session watcher gives you an RStudio-style **Workspace/Variables** view of your live session, lets you launch or attach an R process, and feeds the plot and data viewers. That covers most of what Positron offers for R – Positron is simply more polished, its [Data Explorer](#choosing-between-rstudio-positron-and-vs-code) especially. Posit's [Air](https://posit-dev.github.io/air/) (a fast R formatter and language server) installs as [its own extension](https://marketplace.visualstudio.com/items?itemName=Posit.air-vscode); it bundles its own binary, and so there is nothing else to set up (it comes pre-installed in [Positron](#purpose-built-r-ides-rstudio-and-positron)).

2. **Plots.** Once the R session is attached (see the troubleshooting tip below), plots render automatically in a tab beside your editor through the extension's built-in PNG viewer – no extra setup, because plot capture is part of the session watcher. For a more interactive viewer that resizes, zooms, keeps a plot history, and follows your light/dark theme like RStudio's pane, there are two solid options:

   - [`httpgd`](https://nx10.dev/httpgd/articles/b01_vscode.html) upgrades the extension's built-in viewer in place. Install it in R with `install.packages("httpgd")`, and then enable the `r.plot.useHttpgd` setting (Cmd/Ctrl+`,` opens Settings, where you can search for it). **Restart the R session afterward:** the graphics device is chosen when the session attaches, and so the toggle does nothing until you start a fresh R terminal. Plots then open as a resizable SVG tab beside your editor.

   - For the closest thing to RStudio's Plots pane, the third-party [R Plot Pro extension](https://marketplace.visualstudio.com/items?itemName=ofurkancoban.r-plot-pro) lives in the bottom panel beside the terminal and wraps plots in familiar chrome: a toolbar for paging back and forth through plot history, zooming, and saving or exporting, plus a thumbnail history strip down the side. RStudio migrants tend to find it the most recognizable of the options.

3. **AI layer.** VS Code ships its own **Chat**, and GitHub Copilot is one of the providers it can run on rather than the feature itself, and so pointing Chat at [a key of your own](#bring-your-own-key-byok) works without a Copilot plan. Beyond that, layer on whichever assistant you want as an extension – [Claude Code](https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code) or [Codex](https://marketplace.visualstudio.com/items?itemName=openai.chatgpt) – both searchable under the Extensions pane. Google is the exception: its Gemini Code Assist *extension* and Gemini CLI stop serving individuals [on June 18, 2026](https://developers.google.com/gemini-code-assist/resources/release-notes), and the successor, [Antigravity](https://antigravity.google/), is not another extension but a separate agentic IDE (a Cursor-style VS Code fork; see [choosing an editor](#choosing-between-rstudio-positron-and-vs-code)) – so the bolt-on path for Gemini in VS Code is effectively ending.

> [!TIP]
> The quickest way to start an R session is to click the **R: (not attached)** indicator in VS Code's bottom status bar; it should launch R in a new terminal and switch to **R [VERSION]** in the status bar.
>
> If the status bar remains **not attached** and you see `could not find function '.vsc.attach'` in the R session started in the terminal, you may be seeing the symptoms of a [known issue](https://github.com/REditorSupport/vscode-R/issues/1696) stemming from a change in R 4.6.0. To fix this issue, you can edit your `.Rprofile` file to ensure that `.vsc.attach` is properly defined when R loads. To do so, open your `.Rprofile` with `usethis::edit_r_profile()` (which finds the right file on Windows, macOS, and Linux), and then add this block **at the very end of the file:**
>
> ```r
> # Place at very end of .Rprofile to mimic the startup behavior of R versions before 4.6.0
> if (interactive() && Sys.getenv("RSTUDIO") == "" && Sys.getenv("TERM_PROGRAM") == "vscode") {
>   # find vscode-R's init.R
>   init_path <- file.path(
>     Sys.getenv(if (.Platform$OS.type == "windows") "USERPROFILE" else "HOME"),
>     ".vscode-R", "init.R"
>   )
>   source(init_path)   # run vscode-R's init.R, which places .vsc.* definitions in .First.sys
>   .First.sys()        # ensures .vsc.*() exist (R 4.6.0 no longer auto-fires .First.sys)
> }
> ```
>
> See the [vscode-R issue thread](https://github.com/REditorSupport/vscode-R/issues/1696#issuecomment-4343488948) for context and discussion.

### Choosing between RStudio, Positron, and VS Code

Positron and RStudio are both from Posit and install side by side; Posit maintains both and has set no deprecation timeline for RStudio. Positron is under much heavier active development and wins on mixed R and Python work, multiple R versions and concurrent sessions, the Data Explorer, the VS Code extension marketplace, remote sessions, and running the interpreter separately from the IDE (so a crash in R need not take the IDE down); it also opens Jupyter notebooks natively with full R and Python kernels, which RStudio does not do at all. RStudio still leads on Shiny and package development. Posit's own [feature comparison](https://positron.posit.co/migrate-rstudio-compare.html) weighs the two in detail.

Plain VS Code sits apart from both: it is the most flexible host for AI extensions but the least specialized for R out of the box, because the data-science panes are something you [assemble yourself](#general-purpose-editors-you-assemble-vs-code). It makes the most sense when R is one language among several in an editor you already live in, or when habit or institutional policy fixes your editor and you would rather extend it than adopt a dedicated IDE. For an R-first user, though, Positron is itself a pre-assembled VS Code fork, and so it delivers most of what you would otherwise build by hand with less friction – which is why it, and not plain VS Code, is the default recommendation here. A common pattern among experienced users is to keep more than one installed and open whichever fits the task.

> [!NOTE]
> **The agentic VS Code forks.** Cursor, Windsurf, and Google's [Antigravity](https://antigravity.google/) (the successor to Gemini Code Assist) are VS Code forks tuned for agent-first general development rather than R. They can usually install the R extension from the [Open VSX registry](https://open-vsx.org/) these forks use, but that support is less proven, which makes them the least R-specialized editors covered here. Antigravity's free tier is appealing for students but has tightened since launch; check current limits before relying on it.

> [!IMPORTANT]
> Putting an AI assistant in your editor is not the same as giving it your R session, and which one you get depends on the harness rather than on the editor. [Posit Assistant](#posit-assistant) runs inside the process hosting R, and so it reads your data frames and fitted objects directly. Everything else you might install – GitHub Copilot, a Claude Code or Codex extension, VS Code's own chat, or an agent in a terminal – works from your files and editor context and infers the rest from your code unless you wire up [the bridge in section 4](#4-connecting-an-assistant-to-your-live-r-session). The editor and the bridge are complementary, not alternatives.

### Where R runs in the web browser: cloud options

Browser-driving agents pair naturally with R that already lives in the browser, and cloud R is useful on its own when you want to avoid a local install or give students a uniform environment.

- [Posit Cloud](https://posit.cloud/) gives you a hosted RStudio environment in the browser, with shareable projects, which is useful for teaching and collaboration.
- [Google Colab](https://colab.research.google.com/) supports R two ways: switch the runtime to R under *Runtime > Change runtime type*, or stay in a Python runtime and use the `rpy2` magics (`%load_ext rpy2.ipython`, then `%%R` cells) to mix R and Python. The R runtime resets when you reopen a notebook, and so package installs must be rerun; Drive mounting is also limited in the R runtime.

> *Aside:* [WebR](https://docs.r-wasm.org/webr/latest/) is R compiled to WebAssembly, and so it runs entirely in the browser with no server. It is excellent for *embedding* R into web pages and interactive teaching materials, but it is a poor host for an agentic coding loop. Use it to ship R-backed widgets, not to write code with an assistant.

## 3. Choosing an assistant: model and harness

An assistant is a **model** plus a **harness**: the model generates, and the harness gathers your files, your prompts, and its output, and decides what the model is allowed to touch. The harness determines what the assistant can *do*; the model determines how well it does it, and who bills you. The subsections below are the harness families, roughly from the most R-specific to the most general.

### Posit Assistant

[Posit Assistant](https://assistant.posit.co/) is the harness built into RStudio and Positron, and the only one covered here that runs *inside* the process hosting your R session. It therefore reads your loaded packages, your environment, and your actual data frames with nothing to wire, and it can edit and run code on your behalf as well.

Installing it depends on the editor:

- **In RStudio** (2026.04.0 or later), click the **Posit Assistant** toolbar button and then **Install Posit Assistant** to fetch the add-in.
- **In Positron** (2026.07 or later), it is already there.

Then pick a provider from the **LLM Providers** dialog, on the welcome screen or from the gear menu in the Assistant pane:

- **[Posit AI](https://docs.posit.co/posit-ai/)** is Posit's own managed service, and the fewest steps of anything in this guide: sign in once, with no key to obtain, store, or rotate, and a zero-data-retention agreement with the model vendor. It is a subscription of its own, separate from any existing Claude or ChatGPT plan, and it buys a credit allowance rather than unlimited use: $20 a month including $15 of credits, after a one-time $5 trial. Posit currently offers no academic discount.

- **Any other provider** puts you on [bring your own key](#bring-your-own-key-byok). Anthropic, OpenAI, Google Gemini, OpenRouter, DeepSeek, AWS Bedrock, Google Vertex AI, Microsoft Foundry, Snowflake Cortex, Databricks, local Ollama, and local LM Studio each have an entry, and **OpenAI compatible** takes a base URL, which is how to point the assistant at a university or employer gateway.

> [!CAUTION]
> **In Positron, make sure you are in the right pane.** Positron keeps VS Code's own **Chat** alongside Posit Assistant, and their sidebar icons sit next to each other and open panels that look much the same. Only Posit Assistant has the R session tooling, and only Posit Assistant can use Posit AI. VS Code's Chat in Positron takes your own key from a similar-looking provider list (Anthropic, Azure, Google, Ollama, OpenAI, OpenRouter, xAI, and a custom endpoint), but it reaches your session only through the [MCP server](#r-as-an-mcp-server-mcptools-and-btw) in section 4. If the assistant cannot see your variables, the pane is the first thing to check.

Whichever provider you choose, the model must be **good at tool calling** because that is the mechanism the assistant uses to reach your session. Posit warns that many models have too little tool-calling ability to drive the assistant well, and advises against running a local model in the same session as your work. A weak model will connect and then use the tools badly, which looks like the assistant ignoring your data rather than like a configuration error.

### Vendor agents: Claude Code and Codex

An agentic command-line tool reads and edits files across your project, runs commands (`Rscript`, `R CMD check`, tests), reads the output, and iterates. This is the strongest option for multi-file refactors, package work, and "do this whole task" handoffs. Three terminal agents dominate in 2026, all of which support MCP (see [section 4](#4-connecting-an-assistant-to-your-live-r-session)), subagents, and whole-codebase awareness:

- **[Claude Code](https://www.anthropic.com/claude-code)** (Anthropic). The strongest at multi-file reasoning and code quality, with skills and multi-agent orchestration. Installs via `npm`. Premium, through a Claude subscription or the API. Sessions can be handed off to and continued from the Claude desktop and mobile apps.

  - **Desktop alternative:** On Windows and macOS, the [Claude Desktop app](https://claude.com/product/overview) includes a **Code** tab that is a full-featured desktop frontend for Claude Code, including support for skills, customizable configuration, memory, and MCP servers (see [section 4](#r-as-an-mcp-server-mcptools-and-btw)). Whereas the Claude Code interface in the **Code** tab can manage (and edit) files in an associated folder, the default **Chat** tab cannot. It can generate Artifacts that can be downloaded, but otherwise the **Chat** tab (unlike the **Code** tab) is restricted to copy-and-paste access (as in [copy and paste](#copy-and-paste-stand-alone-chat)).

- **[Codex CLI](https://developers.openai.com/codex/cli/)** (OpenAI). Open-source (Apache 2.0), rebuilt in Rust for speed, notably token-efficient, with strong kernel-level sandboxing. Bundled with ChatGPT plans and paired with a macOS desktop app and an IDE extension so you can start in the terminal and continue in the app.

  - **Desktop alternative:** On Windows and macOS, the [Codex Desktop app](https://chatgpt.com/codex/) is a stand-alone, full-featured desktop frontend for OpenAI Codex, including support for skills, customizable configuration, memory, and [MCP servers](#r-as-an-mcp-server-mcptools-and-btw).

- **[Antigravity CLI](https://antigravity.google/)** (Google). The successor to [Gemini CLI](https://github.com/google-gemini/gemini-cli), rebuilt in Go, with a very large context window and a free tier (verify current terms).

  - Antigravity also ships as an agent-first IDE, which is a VS Code *fork* that can import your existing VS Code settings and extensions.
  - Gemini CLI itself stopped serving consumer accounts on June 18, 2026; only organizations on a Gemini Code Assist Standard or Enterprise license can still run it.

Most people end up using more than one for different tasks. The differences in raw code quality matter less than fit with your existing accounts, your budget, and how much autonomy you are comfortable granting.

**Provider-agnostic alternatives.** The three above are each aligned with one vendor's models (Anthropic, OpenAI, Google). A parallel class of open-source agents is instead model-agnostic: [OpenCode](https://opencode.ai/), among others, is a drop-in terminal coding agent for almost any backend – a commercial API, a local or [Ollama Cloud](https://docs.ollama.com/cloud) open-weight model, or an institutional endpoint such as a university-hosted LLM you reach with an API key. It is MCP capable, and so it still benefits from the [section 4](#4-connecting-an-assistant-to-your-live-r-session) bridge. For anyone with a subsidized or self-hosted model, this is often the cheapest and most privacy-preserving way to drive a capable assistant that still sits beside your IDE.

**Running them inside an IDE terminal.** Each of these coding assistants can be run alone, but it has become very common to use the CLI versions to complement the features of a sophisticated Integrated Development Environment (IDE), like the ones discussed in [section 2](#2-choosing-an-editor). Many IDEs now have special extensions that provide a more seamless frontend to operating the coding assistants.

### Bring your own key (BYOK)

Supplying your own model is the only route that can cost nothing, and for many students and staff it is the best-value option available. Four kinds of endpoint are worth separating, and the last is the one people most often do not know they have:

- **A commercial API key** from Anthropic, OpenAI, or Google, billed per token instead of by subscription.
- **A local open-weight model** through [Ollama](https://ollama.com/) or LM Studio. Nothing leaves your machine and nothing is metered, at the cost of running a smaller model.
- **A remote open-weight model** through a hosted endpoint such as [Ollama Cloud](https://docs.ollama.com/cloud) or [OpenRouter](https://openrouter.ai/), when you want a larger model than your laptop can hold.
- **An institutional endpoint.** Many universities and employers now run an LLM gateway, either serving open-weight models on local hardware or proxying vendor models under a site agreement, often free at the point of use. Ask whoever runs your research computing or IT service whether one exists and how to get a key.

These are largely interchangeable because nearly all of them speak the **OpenAI-compatible Chat Completions API**, which reduces the configuration to a base URL and a key. The same pair of values works in [Posit Assistant](#posit-assistant), in VS Code's built-in chat, and in [`ellmer`](#5-using-an-llm-inside-r-code-ellmer) from inside R, and so a provider you set up once serves all three.

Which harness you pair a key with is a separate question:

- **[Posit Assistant](#posit-assistant)** accepts your key directly and keeps its native session access, which makes it the strongest BYOK setup for R specifically.
- **VS Code's built-in chat** accepts a key from most providers and needs no GitHub account or Copilot plan for chat and agent work, though completions still do.
- **[OpenCode](https://opencode.ai/)** is a model-agnostic terminal agent for almost any backend, and because it is MCP capable, it still reaches your session through [section 4](#4-connecting-an-assistant-to-your-live-r-session).

Provider choice is orthogonal to everything else here: it determines who supplies and bills for the model, not how much the assistant can see of your R session.

### Copy and paste (stand-alone chat)

Pasting code and output into a chat window works and needs no setup. Most people do this in [Claude Desktop](https://claude.ai/download), [claude.ai](https://claude.ai/), [ChatGPT](https://chatgpt.com/), or [Gemini](https://gemini.google.com/). It is fine for quick conceptual questions, explaining an error, or sketching an approach. The cost is friction and context loss: you hand-copy column names, sample rows, error messages, and structure, and the model still guesses at things it cannot see. You can cut most of that friction with the [`btw()`](#btw-the-connective-layer) function, which gathers a structured snapshot of your session and copies it to the clipboard, ready to paste into any chat. But know that the rest of this guide exists to remove the copying entirely.

### Agents that drive your browser or desktop

A newer and more automated variant: instead of you ferrying text, an agent operates an application on your behalf. These are genuinely useful but still maturing; treat them as experimental and keep a human in the loop.

- **Browser-operating agents** such as [Perplexity's Comet](https://www.perplexity.ai/comet), [Claude for Chrome](https://www.anthropic.com/news/claude-for-chrome), and Chrome's built-in [Gemini "Auto Browse"](https://blog.google/products-and-platforms/products/chrome/gemini-3-auto-browse/) can read and act inside web pages. Pointed at a browser-based R session (see below), they can edit a cell, run it, and read the result without your copy-paste.
- **Desktop-operating agents.** ChatGPT's [Work with Apps on macOS](https://help.openai.com/en/articles/10119604-work-with-apps-on-macos) can read and edit code in supported editors, including VS Code and its forks (which covers Positron), JetBrains IDEs, and Xcode, as well as Terminal, iTerm, and Warp. OpenAI's newer Codex desktop app goes further, controlling local tools and acting on whatever is on screen. For R specifically, the editor and terminal integrations cover R code open in a VS Code fork like Positron, or an `R` session running in a terminal; the more autonomous screen-control agents can in principle drive other setups too, such as RStudio or the standalone R GUI (R.app on macOS). These earn their keep when your R lives somewhere without a built-in assistant, and so an RStudio user or someone in a bare terminal gains more here than a Positron user, who already has an assistant in the IDE.

> [!CAUTION]
> An agent that can drive your browser or desktop is a much larger security surface than a chat box. It is exposed to prompt injection from any page or file it reads, and it acts with your privileges. Grant access narrowly, review actions before they run, and never point one at sensitive systems unattended.

## 4. Connecting an assistant to your live R session

Much of the real work in R lives in objects that exist only in your running session, and so an assistant that cannot see them is guessing. How it gets that access depends entirely on *where the harness runs*.

**An assistant inside your IDE already has it.** [Posit Assistant](#posit-assistant) runs in the process hosting R; its session tools are built in and there is nothing to configure. This is a property of the harness rather than of the model: any configured provider calls the same tools, provided the model is capable enough to use them.

**An assistant outside your IDE needs a bridge**, and that bridge is the [Model Context Protocol](https://modelcontextprotocol.io/) (MCP). Sources of data or capability act as MCP *servers*; the assistants that use them act as MCP *clients*. Running an MCP server inside R lets an outside agent query your workspace and read your installed-package documentation. Three R packages matter here:

- **R as the server:** expose your R session and tools to an outside agent like Claude Code ([`mcptools`](https://posit-dev.github.io/mcptools/), via [`btw`](https://posit-dev.github.io/btw/)).
- **The connective layer:** [`btw`](https://posit-dev.github.io/btw/) gathers R context and feeds it in either direction.
- **R as the client:** call an LLM from inside your R code ([`ellmer`](#5-using-an-llm-inside-r-code-ellmer)), which is a different activity covered in [section 5](#5-using-an-llm-inside-r-code-ellmer).

Even with Posit Assistant, the MCP server is worth adding: its documentation tools (help pages, vignettes, and package NEWS) are not part of native session access, though you may want to trim overlapping tools so the assistant is not offered two ways to do the same thing.

### The statefulness catch

**The statefulness catch.** A CLI or desktop coding-assistant agent runs as a separate process *alongside* R, not inside it. Left to its own devices, it edits scripts and runs them with `Rscript`, which spins up a fresh session each time, and so it does not automatically see the objects in your interactive session. Hosting the agent inside an IDE does not change this: the R extension's session viewers, plots pane, and Data Explorer serve *you*, not the agent, and so the editor's live session stays invisible to a CLI tool running in its terminal or as an extension. That is fine for a lot of file-level work, but it is the main reason generated R code misreads your data or a fitted object. The fix is to give the agent a way to query your live session, which is what the rest of this section provides. That approach also gives coding assistants access to R documentation, **which greatly helps to reduce hallucinations.**

### R as an MCP server: mcptools and btw

Here R is the thing being called. The [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) lets a coding agent call the tools R exposes. For R that means an agent can **query the live state of your R session** (the data frames and fitted objects in your global environment) and **read the documentation of the packages you actually have installed** (which can reduce coding hallucinations). That directly addresses [the statefulness catch](#the-statefulness-catch) above; instead of guessing function arguments or re-deriving your data, the agent inspects the real thing.

[`mcptools`](https://posit-dev.github.io/mcptools/) is the protocol layer. In practice, you start from [`btw`](https://posit-dev.github.io/btw/), which ships a ready-made server exposing those documentation and session tools. Setup is two one-time steps.

**First, register the server with each coding assistant you use.** Every agent keeps its own MCP configuration, and so this is per-assistant – but only once each. For Claude Code:

```bash
claude mcp add -s user r-btw -- Rscript -e "btw::btw_mcp_server()"
```

Codex follows the same pattern (or add a `[mcp_servers.r-btw]` block to `~/.codex/config.toml`):

```bash
codex mcp add r-btw -- Rscript -e "btw::btw_mcp_server()"
```

**Second, add a session hook to your `.Rprofile`** (once) so each interactive session reaches into your *running* environment instead of the server only spawning fresh, empty ones:

```r
# usethis::edit_r_profile() to open it
if (interactive() && requireNamespace("btw", quietly = TRUE)) {
  btw::btw_mcp_session()
}
```

The `requireNamespace()` guard keeps R startup from erroring on a machine where `btw` is not installed, and you must restart R afterward because the hook only runs at session start. Drop to `mcptools` directly when you want to expose *arbitrary custom R functions* as tools, or to use R as an MCP *client* that connects out to other MCP servers from an `ellmer` chat.

> [!NOTE]
> `mcptools` was formerly `acquaint`. If you see `acquaint::mcp_server()` in older material, the modern equivalent is `btw::btw_mcp_server()`.

> [!WARNING]
> Session access means the agent can read the objects in your global environment. By default `btw`'s tools only *read and describe* your session and your package docs – they do not run arbitrary R code or modify your environment (that requires opting into `btw`'s off-by-default run-R tool). It is still data exposure, though; be deliberate if a session holds sensitive or very large objects.

### btw: the connective layer

[`btw`](https://posit-dev.github.io/btw/) is the glue across everything above; its own tagline is that it "supercharges `ellmer`." It offers three things from one package:

1. **Context to clipboard.** `btw()` describes objects, package or function docs, and files, and copies the result to your clipboard for any external chat, which takes most of the friction out of the [copy-and-paste workflow](#copy-and-paste-stand-alone-chat).

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

3. **Tools for other clients and servers.** `btw_tools()` returns tool objects to register with any `ellmer` chat, and `btw_mcp_server()` is the MCP server used in the section above. Same tools, whether R is the client or the server.

> [!CAUTION]
> `btw`'s optional run-R tool executes model-written code in your global environment with no sandbox and, as of current `shinychat` and `ellmer`, no review-before-execution step. It is off by default for good reason. Enable it only in a throwaway or sandboxed session, never in a publicly reachable app.

### What the bridge adds to each assistant

The native tooling upgrades any assistant from [section 3](#3-choosing-an-assistant-model-and-harness) rather than replacing it. It hands the model your real session state and the documentation of the packages you actually have installed, which removes the failure named at the top of this guide, where the model guesses at column types or invents a function signature. How much that helps scales with how tightly it is wired in.

| Assistant                                                     | What the `btw` bridge adds                                                                                                                                                                                                                                |
| ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A chat window ([claude.ai](https://claude.ai/), ChatGPT)      | `btw()` copies a structured snapshot of objects and package docs to the clipboard: a manual paste with no live link, but far better than hand-copying                                                                                                     |
| [Posit Assistant](#posit-assistant) (RStudio, Positron)       | live session access is already built in, and it can reach help pages by executing `help()` in the session; `btw_mcp_server()` adds named documentation tools that return cleaned help pages, vignettes, and package NEWS without running code to get them |
| An outside agent (Claude Code, Codex, OpenCode, VS Code Chat) | the MCP server **plus** the `btw_mcp_session()` hook is the only route into the running session, and the package docs come with it                                                                                                                        |

**A worked example of using an outside agent.** Say Claude Code is running in Positron's terminal and you ask it to fix a `dplyr` pipeline that errors on a grouping column. Without the bridge, the agent sees only the script: it guesses the column is `group` (it is actually `treatment_grp`), edits the file, runs it with `Rscript`, reads the error, and guesses again – a slow round-trip that fails outright if the data lives only in your interactive session and never as a file the agent can source. With `btw_mcp_session()` active, it instead calls the session tool, reads the data frame's real column names and types, confirms the grouping variable, and makes the correct edit on the first pass. The same holds for a non-converging model: it can read the fitted object's structure and the actual `lmer` call rather than inferring slot names.

The wiring is just the two pieces shown earlier in this section: register `btw_mcp_server()` with Claude Code, and add `btw_mcp_session()` to your `.Rprofile` so the agent reaches the session you are actually working in. Those two steps settle [the statefulness catch](#the-statefulness-catch), which is why this guide pairs an outside agent with the `btw` MCP server rather than running it blind.

## 5. Using an LLM inside R code: ellmer

Use this when you do not want an assistant so much as to *program with* a model: classify free text, extract structured fields, build a domain chatbot, or run a model over many rows. [`ellmer`](https://ellmer.tidyverse.org/) is the core package. It supports many providers (Anthropic, OpenAI, Google, AWS Bedrock, Azure, and more, including locally hosted open-weight models), with streaming, tool/function calling, and structured data extraction. Chat objects are stateful R6 objects, and so each turn builds on the last.

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

## Cross-cutting cautions

- **Verify generated statistics code.** Plausible-looking R is not correct R. Model-selection logic, contrast coding, random-effects structure, and the meaning of a fitted object's slots all reward a careful read. Treat agent output as a fast first draft to react to, not an answer.
- **Mind what the assistant can see, and do.** Live-session access is data exposure; UI-driving and desktop agents go further and act with your privileges, which widens the prompt-injection surface. Decide consciously what each tool can reach before you wire it up.
- **Cost and keys.** Agentic and in-IDE-assistant workflows bill against your API key and scale with context length and turns. For students and self-funded work, this matters; prefer cheaper models for routine tasks, and reserve top-tier models for hard problems.
- **Reproducibility.** If an LLM touched a result that goes into a paper, the pipeline should still run without it. Script the calls and keep a human-readable record of what the model did.

## Use-case quick reference

| Goal                                                    | Reach for                                                                         |
| ------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Ask one-off questions, paste a traceback                | [A stand-alone chat](#copy-and-paste-stand-alone-chat)                            |
| Gather session context to paste into any chat           | [`btw()`](#btw-the-connective-layer)                                              |
| Have an external agent operate R or a cloud session     | [A browser- or desktop-driving agent](#agents-that-drive-your-browser-or-desktop) |
| Run R without a local install                           | [A cloud notebook or IDE](#where-r-runs-in-the-web-browser-cloud-options)         |
| Get inline completions and chat without leaving the IDE | [Posit Assistant](#posit-assistant)                                               |
| Hand off a multi-file task to an agent                  | [A vendor agent](#vendor-agents-claude-code-and-codex)                            |
| Let an agent see your real session and package docs     | [R as an MCP server](#r-as-an-mcp-server-mcptools-and-btw)                        |
| Use a free, subsidized, or self-hosted model            | [Bring your own key](#bring-your-own-key-byok)                                    |
| Call an LLM *from inside* your R code                   | [`ellmer`](#5-using-an-llm-inside-r-code-ellmer)                                  |

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
- Antigravity (CLI and IDE; successor to Gemini CLI): <https://antigravity.google/>
- Posit Assistant: <https://assistant.posit.co/>
- Posit AI documentation: <https://docs.posit.co/posit-ai/>
- OpenCode: <https://opencode.ai/>
- Google Antigravity (successor to Gemini Code Assist / Gemini CLI for individuals): <https://antigravity.google/>
- ChatGPT "Work with Apps" on macOS: <https://help.openai.com/en/articles/10119604-work-with-apps-on-macos>
- rocker (Docker images for R): <https://rocker-project.org/>
- WebR: <https://docs.r-wasm.org/webr/latest/>
- Companion guide: [Using LLMs with MATLAB](using-llms-with-matlab.md)

---

*MIT License. Copyright © 2026 Theodore P. Pavlic. Shared as a starting point; verify against current documentation, because this tooling changes quickly.*
