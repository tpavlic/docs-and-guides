# Using LLMs with MATLAB: A Practical Map of the Approaches

A short field guide to the main ways of putting a large language model to work alongside MATLAB. The quick start comes first; the remaining sections are ordered roughly from *least integrated with your MATLAB session* to *most*. Written for colleagues and students who already know MATLAB and want to choose a setup deliberately rather than by accident.

> **Why MATLAB needs its own guide.** Much of the real work in MATLAB lives in a *stateful, interactive session*: variables in the base workspace, figures you just drew, a model object whose actual structure matters, the toolboxes you happen to have licensed. A generic "edit a file, run it, read the output" loop throws that away, which is why LLMs tend to hallucinate function signatures, invent a toolbox you do not own, or guess at the shape of your data when they cannot see your session. The approaches below differ mostly in how well they close that gap – and MATLAB now ships an official mechanism, the [MATLAB MCP Server](https://github.com/matlab/matlab-mcp-server), for closing it.

> [!NOTE]
> This space moves fast. Versions, product names, and defaults change often. Treat everything below as a starting point and check the linked docs. Last reviewed August 2026.

> [!TIP]
> **TL;DR.** If you just want a recommendation: install a recent MATLAB release, run [Claude Code](https://www.anthropic.com/claude-code) (or another MCP-capable assistant), and register the [MATLAB MCP Server](https://github.com/matlab/matlab-mcp-server) so the agent can run code, check it, run tests, and read your installed toolboxes. The one step people miss is **connecting the server to the MATLAB desktop you are actually using** (via `shareMATLABSession()`) so the agent reads your live workspace rather than spawning a blind, empty session. The [MATLAB Agentic Toolkit](https://github.com/matlab/matlab-agentic-toolkit) can wire most of this up for you. The [quick start](#1-quick-start-llm-integrated-with-a-live-matlab-session) has the exact steps; the rest of the guide is background and alternatives.

---

- [1. Quick start: LLM integrated with a live MATLAB session](#1-quick-start-llm-integrated-with-a-live-matlab-session)
- [2. No-integration case: Keep the coding assistant separate from MATLAB](#2-no-integration-case-keep-the-coding-assistant-separate-from-matlab)
  - [Copy and paste (stand-alone chat)](#copy-and-paste-stand-alone-chat)
  - [Agents that drive your browser or desktop](#agents-that-drive-your-browser-or-desktop)
  - [Where MATLAB runs in the browser: cloud options](#where-matlab-runs-in-the-browser-cloud-options)
- [3. Coding assistant as script editor: Edit .m files without session information](#3-coding-assistant-as-script-editor-edit-m-files-without-session-information)
  - [Bring Your Own Key (BYOK): provider-agnostic agents](#bring-your-own-key-byok-provider-agnostic-agents)
  - [What a file-level agent can and cannot see](#what-a-file-level-agent-can-and-cannot-see)
- [4. Editors and IDEs that simplify use of coding assistants alongside MATLAB](#4-editors-and-ides-that-simplify-use-of-coding-assistants-alongside-matlab)
  - [The MATLAB desktop and MATLAB Copilot](#the-matlab-desktop-and-matlab-copilot)
  - [General-purpose editors you assemble: VS Code](#general-purpose-editors-you-assemble-vs-code)
  - [Choosing between the MATLAB desktop and VS Code](#choosing-between-the-matlab-desktop-and-vs-code)
- [5. Beyond simple editing: Accessing the MATLAB session with coding assistants (Model Context Protocol)](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol)
  - [MATLAB as the server: the MATLAB MCP Server](#matlab-as-the-server-the-matlab-mcp-server)
  - [Installing and registering the server (the bare minimum)](#installing-and-registering-the-server-the-bare-minimum)
  - [Connecting to your running session: session modes](#connecting-to-your-running-session-session-modes)
  - [MATLAB as the client: calling LLMs and building agents in MATLAB](#matlab-as-the-client-calling-llms-and-building-agents-in-matlab)
  - [Force multiplier for any editor: the bridge across the setups above](#force-multiplier-for-any-editor-the-bridge-across-the-setups-above)
- [Cross-cutting cautions](#cross-cutting-cautions)
- [Reproducibility and sandboxing](#reproducibility-and-sandboxing)
- [Use-case quick reference](#use-case-quick-reference)
- [Selected references](#selected-references)

---

## 1. Quick start: LLM integrated with a live MATLAB session

This is the recommended default for most people. **Steps 1–3 are all you need to get the LLM link working**; step 4 is best-practice and special-case additions. Read on past this for the reasoning behind each choice and the alternatives for when this default does not fit.

1. **A recent MATLAB release** as your environment. The MATLAB desktop is itself a capable IDE (editor, workspace browser, debugger, plots). The [MCP Server](https://github.com/matlab/matlab-mcp-server) supports MATLAB R2021a or later; a few niceties want newer releases – plain-text Live Scripts need R2025a+, and the in-product [MATLAB Copilot](https://www.mathworks.com/products/matlab/generative-ai.html) needs R2025a+ *and* an entitlement of its own (see [section 4](#the-matlab-desktop-and-matlab-copilot)). Make sure `matlab` is on your system `PATH`.

2. **[Claude Code](https://www.anthropic.com/claude-code) as the agent** for multi-file, "do this whole task" work – signed in with your Claude subscription, not a metered API key.

   - Run it however you like: as a CLI in any terminal (including the one inside [VS Code](#general-purpose-editors-you-assemble-vs-code)), or via the **Code** tab of the [Claude desktop app](https://claude.com/product/overview), which is what MathWorks' own [walkthrough](https://blogs.mathworks.com/matlab/2025/11/03/exploring-the-matlab-model-context-protocol-mcp-core-server-with-claude-desktop/) uses.
   - Most other MCP-capable assistants (Codex, GitHub Copilot, Antigravity CLI, Amp) work too; only the registration syntax differs.

3. **[MATLAB MCP Server](https://github.com/matlab/matlab-mcp-server)** so the agent can run and check code, run tests, and read your installed toolboxes rather than guessing. Two one-time actions:

   Register the server with your assistant. The fastest path is to let the [MATLAB Agentic Toolkit](https://github.com/matlab/matlab-agentic-toolkit) install and register it for you (it supports Claude Code, Codex, GitHub Copilot, Amp, and Gemini CLI). Or do it by hand once you have the [binary](https://github.com/matlab/matlab-mcp-server/releases/latest):

   ```bash
   claude mcp add --transport stdio matlab -- /full/path/to/matlab-mcp-server
   ```

   **Connect it to the MATLAB you are actually using.** Run the helper once to enable session sharing, then call `shareMATLABSession()` in your desktop session so the agent attaches to *that* workspace instead of spawning a fresh, empty one (full detail in [section 5](#connecting-to-your-running-session-session-modes)):

   ```bash
   /full/path/to/matlab-mcp-server --setup-matlab
   ```

   ```matlab
   shareMATLABSession   % run in the MATLAB desktop you want the agent to see
   ```

   That call can also run automatically each time MATLAB launches, by placing it in your `startup.m` – see [Connecting to your running session](#connecting-to-your-running-session-session-modes) for the setup details. If you use a different assistant, the MCP registration is slightly different; ask it to "register the MATLAB MCP Server" and it will usually do it or tell you exactly how.

4. **Recommended and special-case additions** (none are required for the LLM link itself):
   - **[Large Language Models (LLMs) with MATLAB](https://github.com/matlab-deep-learning/llms-with-matlab)** when the model belongs *inside* your MATLAB code (classify text, extract structured fields, call a model over many rows) rather than acting as an external assistant. See [section 5](#matlab-as-the-client-calling-llms-and-building-agents-in-matlab).
   - **[MATLAB Copilot](https://www.mathworks.com/products/matlab/generative-ai.html)** (R2025a+, licensed separately from MATLAB) for in-editor completions and a docs-grounded chat without leaving the desktop. See [section 4](#the-matlab-desktop-and-matlab-copilot).
   - **[Simulink Agentic Toolkit](https://github.com/matlab/simulink-agentic-toolkit)** / Simulink MCP server when your work is model-based and you want the agent to read and edit Simulink models too.

## 2. No-integration case: Keep the coding assistant separate from MATLAB

In this category the LLM runs as its own application, completely outside your MATLAB toolchain, and the two are bridged either by **you** (copy and paste) or by **UI automation** (an agent that operates your screen). It is the least integrated family of approaches, which makes it the easiest to start with and a dependable fallback when nothing fancier is set up. The tradeoff is that the assistant has little or no native understanding of MATLAB's internals; it sees text or pixels, not your workspace variables or the toolboxes you own.

### Copy and paste (stand-alone chat)

Pasting code and output into a chat window works and needs no setup. Most people do this in [Claude Desktop](https://claude.ai/download), [claude.ai](https://claude.ai/), [ChatGPT](https://chatgpt.com/), or [Gemini](https://gemini.google.com/). It is fine for quick conceptual questions, explaining an error, or sketching an approach. The cost is friction and context loss: you hand-copy variable names, sizes, the error message, maybe a `whos` dump, and the model still guesses at things it cannot see – including which toolboxes you actually have, and so it may cheerfully suggest a function you are not licensed for. The rest of this guide exists to remove the copying entirely.

### Agents that drive your browser or desktop

A newer and more automated variant: instead of you ferrying text, an agent operates an application on your behalf. These are genuinely useful but still maturing; treat them as experimental and keep a human in the loop.

- **Browser-operating agents** such as [Perplexity's Comet](https://www.perplexity.ai/comet), [Claude for Chrome](https://www.anthropic.com/news/claude-for-chrome), and Chrome's built-in [Gemini "Auto Browse"](https://blog.google/products-and-platforms/products/chrome/gemini-3-auto-browse/) can read and act inside web pages. Pointed at [MATLAB Online](https://matlab.mathworks.com/) (see below), they can edit code in a Live Script, run it, and read the result without your copy-paste.
- **Desktop-operating agents.** ChatGPT's [Work with Apps on macOS](https://help.openai.com/en/articles/10119604-work-with-apps-on-macos) can read and edit code in supported editors, and OpenAI's Codex desktop app goes further, acting on whatever is on screen. For MATLAB specifically, these can drive the MATLAB desktop or a VS Code window holding `.m` files, which is the main way to get an external agent "into" the desktop MATLAB UI before the [MCP server](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol) existed.

> [!CAUTION]
> An agent that can drive your browser or desktop is a much larger security surface than a chat box. It is exposed to prompt injection from any page or file it reads, and it acts with your privileges. Grant access narrowly, review actions before they run, and never point one at sensitive systems unattended.

### Where MATLAB runs in the browser: cloud options

Browser-driving agents pair naturally with MATLAB that already lives in the browser, and cloud MATLAB is useful on its own when you want to avoid a local install or give students a uniform environment.

- [MATLAB Online](https://matlab.mathworks.com/) runs the full desktop in a browser tab, tied to your MathWorks license, with the Live Editor, file storage, and most toolboxes. It is the natural target for a browser-operating agent and for teaching.
- [MATLAB Mobile](https://www.mathworks.com/products/matlab-mobile.html) and the cloud-backed [MATLAB Drive](https://www.mathworks.com/products/matlab-drive.html) round out the hosted options for lightweight access and file sync.

> *Aside:* The MCP integration in [section 5](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol) targets a **locally installed** MATLAB that the server can launch or attach to. Browser-based MATLAB Online is excellent for access and teaching but is not where the MCP loop runs; for the agentic workflow, install MATLAB locally.

## 3. Coding assistant as script editor: Edit .m files without session information

An agentic command-line tool reads and edits files across your project, runs commands (`matlab -batch`, tests), reads the output, and iterates. This is the strongest option for multi-file refactors, larger code bases, and "do this whole task" handoffs. The terminal agents that dominate in 2026 all support MCP (see [section 5](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol)):

- **[Claude Code](https://www.anthropic.com/claude-code)** (Anthropic) – the strongest at multi-file reasoning and code quality, with skills and multi-agent orchestration; signed in through a Claude subscription or the API. On Windows and macOS the [Claude desktop app](https://claude.com/product/overview) has a **Code** tab that is a full desktop frontend for it (the one used in MathWorks' [walkthrough](https://blogs.mathworks.com/matlab/2025/11/03/exploring-the-matlab-model-context-protocol-mcp-core-server-with-claude-desktop/)).
- **[Codex CLI](https://developers.openai.com/codex/cli/)** (OpenAI) – open-source, rebuilt in Rust, token-efficient, with strong sandboxing, plus a macOS desktop app and IDE extension.
- **[Antigravity CLI](https://antigravity.google/)** (Google) – the successor to [Gemini CLI](https://github.com/google-gemini/gemini-cli), rebuilt in Go, with a very large context window and a free tier (verify current terms).
  - Antigravity also ships as an [agent-first IDE](https://antigravity.google/download), which is a VS Code *fork* that can import your existing VS Code settings and extensions.

This guide deliberately does **not** cover installing these tools; each has its own quick start.

### Bring Your Own Key (BYOK): provider-agnostic agents

The three agents above are each aligned with one vendor's models (Anthropic, OpenAI, and Google) and are normally driven by that vendor's subscription. A parallel class of open-source agents is instead model-agnostic: [OpenCode](https://opencode.ai/), among others, is a drop-in terminal coding agent for whatever backend you hold a key for. Four kinds of backend are worth separating, and the last two are the ones students most often do not know they have:

- **A commercial API key** from Anthropic, OpenAI, or Google, billed per token instead of by subscription. This is the flexible option and the easiest one to run up a bill with.
- **A local open-weight model** through [Ollama](https://ollama.com/). Nothing leaves your machine and nothing is metered, at the cost of running a smaller model than the frontier ones.
- **A remote open-weight model** through a hosted endpoint such as [Ollama Cloud](https://docs.ollama.com/cloud) or [OpenRouter](https://openrouter.ai/), when you want a larger open-weight model than your laptop can hold.
- **An institutional endpoint.** Many universities and employers now run their own LLM gateway, either serving open-weight models on local hardware or acting as a proxy to vendor models under a site agreement – **often free at the point of use** and frequently the best-value option a student has. Ask whoever runs your research computing or IT service whether one exists and how to get a key.

These backends are largely interchangeable because nearly all of them speak the **OpenAI-compatible Chat Completions API**, which reduces the configuration to a base URL and a key. That same pair of values usually works in a terminal agent, in [VS Code's chat](#general-purpose-editors-you-assemble-vs-code), and in [calling LLMs from inside MATLAB itself](#matlab-as-the-client-calling-llms-and-building-agents-in-matlab), and so a provider you set up once serves all three.

Provider choice is orthogonal to everything else in this guide. It determines who supplies and bills for the model, not how much the assistant can see of your MATLAB session, and these agents are MCP capable, and so a bring-your-own-key setup still gets the full [section 5](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol) bridge to your live workspace. For anyone with a subsidized or self-hosted model, this is often the cheapest and most privacy-preserving way to drive a capable assistant beside your editor.

### What a file-level agent can and cannot see

**The statefulness catch.** A CLI agent runs as a separate process *alongside* MATLAB, not inside it. Left to its own devices, it edits `.m` files and runs them with `matlab -batch`, which starts a fresh MATLAB process each time – an empty base workspace, no figures, none of the variables you have built up interactively. Hosting the agent in an editor does not change this: the MATLAB desktop's Workspace browser, Variables view, and Figures serve *you*, not the agent. That is fine for a lot of file-level work, but it is the single biggest reason generated MATLAB code misreads your data, assumes the wrong array shape, or calls into a toolbox you do not have. The fix is to give the agent a way to run code in – and read the state of – your *running* session, which is what [section 5](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol) provides. That same bridge also exposes MATLAB's coding guidelines and lets the agent enumerate your installed toolboxes, **which greatly helps to reduce hallucinations.**

## 4. Editors and IDEs that simplify use of coding assistants alongside MATLAB

These live inside an editor and can see your files and editor context. Most of them reason from your *open files and project*, not from the variables in your *running* MATLAB session. An assistant can read a script that fits a model, but it does not, by default, see the fitted object sitting in your workspace or the figure on your screen. Closing that gap is what [section 5](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol) is for.

### The MATLAB desktop and MATLAB Copilot

The MATLAB desktop is the purpose-built environment: editor, Live Editor, Workspace browser, debugger, and Plots pane in one place, with every toolbox you own integrated. Its native AI layer is **[MATLAB Copilot](https://www.mathworks.com/products/matlab/generative-ai.html)**, a generative-AI assistant introduced in **R2025a**. It offers a docs-grounded chat (answers sourced from MathWorks documentation and examples), inline completions and natural-language-to-code in the Editor, and help that explains unfamiliar code, clarifies error messages, and generates tests with MATLAB Test. Because its answers are grounded in MathWorks documentation, it is well suited to MATLAB-idiomatic questions where a general assistant would guess.

Copilot is an *in-editor assistant*, though: like the IDE assistants in any language, it works primarily from your code and the documentation, not from the live contents of your base workspace. For agentic, multi-file work – and to let any external agent read your actual session – pair the desktop with an [agentic CLI (or Desktop app)](#3-coding-assistant-as-script-editor-edit-m-files-without-session-information) and the [MCP server](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol).

> [!IMPORTANT]
> **Check whether you actually have MATLAB Copilot before planning around it.** It is licensed separately from MATLAB rather than bundled with it: there is nothing to install, and access is an entitlement attached to your MathWorks Account. MathWorks' [FAQ](https://www.mathworks.com/products/matlab-copilot/faq.html) lists it as included with campus-wide access and Institute-Wide Licenses, which is how most university users will get it, and as purchasable individually by commercial and academic customers, but not available with the Student and Home offerings. A student on a personally bought Student Suite may therefore have MATLAB and no Copilot, whereas a classmate working under the same campus agreement has both. It further requires R2025a or newer, a signed-in MathWorks Account, and an internet connection, and it is unavailable in some regions. MathWorks' FAQ also indicates that Copilot moves onto the same license as MATLAB as of R2026b; confirm your own entitlement rather than assuming it either way. If you do not have it, a [bring-your-own-key](#bring-your-own-key-byok-provider-agnostic-agents) assistant covers much of the same ground at no cost, and the [MCP server](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol) is free and independent of any of this.

### General-purpose editors you assemble: VS Code

[VS Code](https://code.visualstudio.com/) is the most flexible host for AI extensions – GitHub Copilot's agent mode, [Claude Code](https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code), and [Codex](https://marketplace.visualstudio.com/items?itemName=openai.chatgpt) all ship as extensions – so it is a natural home if MATLAB is one language among several you already edit there. To make it a real MATLAB environment rather than a plain text editor, add two pieces:

1. **Language support and code execution.** Install the official **[MATLAB extension for Visual Studio Code](https://github.com/mathworks/MATLAB-extension-for-vscode)** ([marketplace](https://marketplace.visualstudio.com/items?itemName=MathWorks.language-matlab)). On top of syntax highlighting, code analysis, and navigation, it connects to a locally installed MATLAB to **run a whole file or the current selection** in an integrated MATLAB terminal and provides full **debugging** – breakpoints, step controls, and workspace variables, watchpoints, and the call stack in the Run and Debug view. It also discovers and runs MATLAB unit tests through VS Code's Test Explorer, and it can open and create MATLAB Projects. Basic editing works with no MATLAB installed; the advanced features need **MATLAB R2021b or later** on your `PATH` (R2021a is no longer supported), and the separate MATLAB view that lets you browse and rename base-workspace variables outside a debug session needs R2023a or later. See MathWorks' [overview](https://www.mathworks.com/products/matlab-with-vs-code.html).

2. **Plots and inline output.** A figure your code draws opens in a **separate MATLAB figure window**, the same as from the desktop, rather than docked in the editor – and you can type `desktop` in the MATLAB terminal at any point to bring up the full MATLAB desktop. To keep figures and results *inline* instead, run MATLAB through a Jupyter notebook inside VS Code: add the Jupyter extension and [MATLAB Integration for Jupyter](https://github.com/mathworks/jupyter-matlab-proxy) (the `jupyter-matlab-proxy` package plus its MATLAB kernel), after which MATLAB cells render text and plots beneath each cell. MathWorks documents the VS Code steps in [its guide](https://github.com/mathworks/jupyter-matlab-proxy/blob/main/install_guides/vscode/README.md).

3. **AI layer.** Layer on whichever assistant you want as an extension: Claude Code or Codex, each searchable in the Extensions pane. VS Code also ships its own **Chat**, a capable agent in its own right, on which GitHub Copilot is one provider among several rather than the feature itself. It accepts your own key, which makes it the natural companion to the [bring-your-own-key](#bring-your-own-key-byok-provider-agnostic-agents) setups in section 3.

   Run the `Chat: Manage Language Models` command to register a provider. Anthropic, OpenAI, Gemini, Azure, Hugging Face, and OpenRouter are built in, alongside local runtimes such as Ollama and Foundry Local and any endpoint that speaks the Chat Completions, Responses, or Messages APIs – which is what lets an institutional gateway drop straight in. Chat, agent mode, and **any MCP servers you have registered**, including the MATLAB one from [section 5](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol), then run against that model, billed by your provider rather than against a Copilot quota.

   This path needs **no GitHub account and no Copilot plan**: once at least one such model is configured, VS Code opens the Chat view and stops prompting you to sign in. The exceptions are inline completions, next-edit suggestions, and semantic search, which still require a GitHub sign-in because bring-your-own-key covers chat and agent work rather than as-you-type completion. For a student with an institutional key and no Copilot or Claude subscription, this is the shortest route to a full-featured, MCP-capable agent sitting beside a real MATLAB session. On a managed Copilot Business or Enterprise account, though, an administrator can disable the feature by policy.

The catch, exactly as in [section 3](#3-coding-assistant-as-script-editor-edit-m-files-without-session-information), is that the AI extension and the MATLAB extension are separate: the assistant does not automatically see the session the MATLAB extension is driving. The [MCP server](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol) bridges them – and it does more than bridge. Because the server can launch and drive its own MATLAB (in `desktop` or `nodesktop` mode), an agent can run code and open figures on demand; it is execution-first rather than a read-only window onto your session. In practice, you can leave a lightweight MATLAB running in the background and let the agent generate and refresh graphics while you stay in your editor – the arrangement [section 5](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol) builds toward.

### Choosing between the MATLAB desktop and VS Code

For most MATLAB-first work the desktop is the path of least resistance: every toolbox, the Live Editor, and now MATLAB Copilot are already there, with nothing to assemble. Reach for VS Code when MATLAB is one language among several you already live in, when you want a particular agentic extension (Claude Code, Codex) in the same window as your editor, or when institutional habit fixes your editor. A common pattern among experienced users is to keep both open: the desktop for interactive analysis and figures, VS Code (or a terminal agent) for multi-file refactors – with the [MCP server](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol) connecting whichever agent you choose back to the one running MATLAB session.

> [!IMPORTANT]
> Integrating agentic AI into an editor is not the same as giving that AI your MATLAB session. In the MATLAB desktop, in VS Code, and in any agentic VS Code fork (Cursor, Windsurf, [Antigravity](https://antigravity.google/)) alike, the built-in assistants work primarily from your files and editor context; by default they do not see the variables in your *running* workspace or the figures on your screen, and instead infer structure from your code and guess the rest. To let any of these tools run code in – and read the state of – your real session, wire up the [native bridge in section 5](#5-beyond-simple-editing-accessing-the-matlab-session-with-coding-assistants-model-context-protocol) as well. The editor and the bridge are complementary, not alternatives.

## 5. Beyond simple editing: Accessing the MATLAB session with coding assistants (Model Context Protocol)

In modern agentic AI, different agents and resources talk to each other through the Model Context Protocol (MCP). Sources of data or capability act as MCP *servers*; the assistants that use them act as MCP *clients*. A coding assistant on its own can only edit `.m` files; adding an MCP server lets it actually run MATLAB, check code, run tests, and inspect your environment. MATLAB sits on **both** sides of this protocol:

- **MATLAB as the server:** expose your MATLAB session and tools to an outside agent like Claude Code ([MATLAB MCP Server](https://github.com/matlab/matlab-mcp-server)).
- **MATLAB as the client:** call an LLM from inside your MATLAB code ([LLMs with MATLAB](https://github.com/matlab-deep-learning/llms-with-matlab)).

### MATLAB as the server: the MATLAB MCP Server

Here MATLAB is the thing being called. The [MATLAB MCP Server](https://github.com/matlab/matlab-mcp-server) is MathWorks' official MCP server. It exposes a focused set of tools that an agent can call:

- **`detect_matlab_toolboxes`** – list installed MATLAB and toolbox versions, so the agent only suggests functions you can actually run.
- **`check_matlab_code`** – static analysis of a `.m` file (style issues, deprecated functions, performance concerns).
- **`evaluate_matlab_code`** – execute MATLAB code and return the output.
- **`run_matlab_file`** – execute a `.m` script.
- **`run_matlab_test_file`** – run a MATLAB unit-test file and return structured results.

It also serves two **resources** – `guidelines://coding` (MATLAB coding standards) and `guidelines://plain-text-live-code` (rules for generating plain-text Live Scripts, R2025a+) – so the agent writes MATLAB-idiomatic code and version-control-friendly Live Scripts rather than improvising a house style. A raw `.mlx` is a zipped bundle of XML and images that a model cannot author blind, and early adopters had to hand-teach it the format (see this [community write-up](https://www.mathworks.com/matlabcentral/discussions/ai/885947-experiments-with-claude-code-and-matlab-mcp-core-server)). The plain-text `.m` Live Code format plus this resource let the agent generate formatted text, LaTeX, interactive controls, and embedded figures directly.

The built-in tool list is not the limit. Passing `--extension-file` a JSON file that describes your own MATLAB functions exposes them as MCP tools beside the five above, and multiple files are allowed. For a group with standard analysis routines, this promotes them to things the agent calls by name rather than reconstructs from your source each session; the [custom tools guide](https://github.com/matlab/matlab-mcp-server/blob/main/guides/custom-tools.md) in the repository has the schema.

The agent can **run code in and inspect the real state of your MATLAB session** instead of guessing, and it can **see which toolboxes you actually have** rather than hallucinating one. That directly addresses the statefulness catch from [sections 3](#3-coding-assistant-as-script-editor-edit-m-files-without-session-information) and [4](#4-editors-and-ides-that-simplify-use-of-coding-assistants-alongside-matlab).

> [!NOTE]
> **Simulink, too.** The [Simulink Agentic Toolkit](https://github.com/matlab/simulink-agentic-toolkit) builds on this same MCP server and extends it into Model-Based Design. On top of the MATLAB tools above, it adds Simulink-specific ones for reading model structure, editing blocks and connections, querying parameters, and running simulations, plus skills spanning Simulink, Stateflow, Simscape, System Composer, and requirements traceability. If your work is models rather than scripts, install it alongside (or instead of) the plain MATLAB toolkit; the registration and live-session story are the same as above. Separately from the MCP route, R2026a also added **Simulink Copilot** for Model-Based Design and **Polyspace Copilot** for code analysis, in-product assistants in the same family as [MATLAB Copilot](#the-matlab-desktop-and-matlab-copilot) and subject to the same licensing question.

### Installing and registering the server (the bare minimum)

The [project README](https://github.com/matlab/matlab-mcp-server) is the authoritative install guide; this is the short version for someone who already understands MCP servers and just needs the key points.

1. **Prerequisites.** A locally installed MATLAB **R2021a or later**, with `matlab` on your system `PATH`.

2. **Get the binary.** Download the release for your OS from the [releases page](https://github.com/matlab/matlab-mcp-server/releases/latest) – the assets are named `matlab-mcp-server-windows-x64.exe`, `matlab-mcp-server-macos-arm64`, `matlab-mcp-server-macos-x64`, and `matlab-mcp-server-linux-x64` – or build it with Go:

   ```bash
   go install github.com/matlab/matlab-mcp-server/cmd/matlab-mcp-server@latest
   ```

   On macOS you may need to clear the quarantine flag and mark it executable (`chmod +x`) after downloading.

3. **Register it with your assistant.** For Claude Code:

   ```bash
   claude mcp add --transport stdio matlab -- /full/path/to/matlab-mcp-server
   ```

   Codex, GitHub Copilot (a `.vscode/mcp.json` entry), Amp, and Gemini CLI follow each tool's usual MCP-registration pattern. **Claude Desktop** skips the command line: download the `matlab-mcp-server.mcpb` bundle from the same releases page, double-click it to install the server as a desktop extension, and set any arguments under *Settings > Extensions > Configure*. Enable Anthropic's Filesystem extension alongside it, scoped to your project folder, or Claude will not be able to read and write the files it asks MATLAB to run. The easiest route for any of them is the **[MATLAB Agentic Toolkit](https://github.com/matlab/matlab-agentic-toolkit)**: clone its repo, open your agent in that folder, and ask it to "set up the toolkit," after which a setup skill detects MATLAB, installs the server, and registers it. Its automatic configuration currently covers Claude Code, GitHub Copilot, Codex, Gemini CLI, and Amp; Antigravity CLI is not yet on that list; register the server by hand there. The toolkit is more than an installer, though – alongside the server it bundles a curated set of **agent skills**, Markdown instruction files that teach the agent MATLAB engineering practices and recommended workflows, either invoked explicitly (a slash command in Claude Code) or pulled in automatically when a task calls for one.

Useful flags include `--matlab-root` (point at a specific MATLAB install), `--matlab-display-mode` (`desktop` vs `nodesktop`), `--initial-working-folder`, `--initialize-matlab-on-startup=true` (start MATLAB with the server instead of waiting for the first tool call), and `--matlab-session-mode` (covered next). When the server misbehaves, `--log-level` (`debug`, `info`, `warn`, or `error`) and `--log-folder` are where to start. Run with `--disable-telemetry=true` to opt out of anonymized data collection.

Every flag also has an environment-variable form: uppercase the name, replace hyphens with underscores, and prefix `MW_MCP_SERVER_`, and so `--matlab-root` becomes `MW_MCP_SERVER_MATLAB_ROOT`. That is often easier to set in an assistant's configuration file than a command line, and CLI flags win where you set both.

> [!TIP]
> **Give the agent and MATLAB a shared working folder.** A common early stumble is the agent writing a `.m` file to *its own* sandbox (some assistants run with a private filesystem) and then asking your local MATLAB to run a path it cannot see, which sends it into a retry loop. Both MathWorks' [walkthrough](https://blogs.mathworks.com/matlab/2025/11/03/exploring-the-matlab-model-context-protocol-mcp-core-server-with-claude-desktop/) and a [community write-up](https://www.mathworks.com/matlabcentral/discussions/ai/885947-experiments-with-claude-code-and-matlab-mcp-core-server) hit this. The fix is to pin a real folder on your machine – pass `--initial-working-folder /path/to/work`, scope any filesystem access to the same place, and name that folder explicitly in your prompts so the agent reads and writes where MATLAB is actually looking.

> [!NOTE]
> **The old name still turns up.** Until the June 2026 rename this was the MATLAB MCP **Core** Server, and the repository, binary, and Go module were all `matlab-mcp-core-server`. The old GitHub URLs redirect, but earlier write-ups, including the MathWorks blog posts and community threads cited in this guide, still use the former name and the old binary path, and so adjust any command you copy from them.

### Connecting to your running session: session modes

This is the step that turns the server from "an agent that runs MATLAB in the dark" into "an agent that works in the session you are looking at," and it is the MATLAB analog of the live-session bridge that makes this whole approach worthwhile. The behavior is governed by `--matlab-session-mode`:

- **`auto`** (default) – attach to a shared running session if one is available, otherwise start a new one.
- **`existing`** – attach only to a session you have explicitly shared (it will not start one).
- **`new`** – always start a fresh session.

Session sharing needs **MATLAB R2023a or later**, a higher floor than the R2021a the server itself supports. To make a session attachable, run the one-time setup, which installs an add-on called MATLAB MCP Server Toolbox, and then share the desktop session you want the agent to use:

```bash
/full/path/to/matlab-mcp-server --setup-matlab
```

```matlab
shareMATLABSession   % run in the MATLAB desktop holding your variables
```

With a session shared this way, `auto` (or `existing`) attaches to *that* workspace: the agent's `evaluate_matlab_code` calls run against your real variables, your loaded data, and your licensed toolboxes, and you can watch the commands execute in the desktop. Without it, the agent gets a fresh, empty session every time and is back to guessing – the exact failure this section exists to remove.

> [!TIP]
> **Share automatically at startup.** To avoid running `shareMATLABSession` by hand each session, add it to your `startup.m` – the user script MATLAB runs at launch, the rough analog of R's `.Rprofile`. Open (or create) the right file on any platform with `edit(fullfile(userpath, 'startup.m'))` because `userpath` is always on the search path, and guard the call so it fires only in an interactive desktop and only once the MCP setup has installed the helper:
>
> ```matlab
> % in startup.m
> if usejava('desktop') && exist('shareMATLABSession', 'file')
>     shareMATLABSession
> end
> ```
>
> Restart MATLAB for it to take effect, and run `which startup` to confirm which file MATLAB will actually execute because it runs only the first `startup` on the path. Leave `matlabrc` alone (it is reserved for MathWorks and administrators), and do not add a `cd` here – set the start folder through the Initial working folder setting or the server's `--initial-working-folder` instead.

> [!WARNING]
> Session access cuts both ways. `evaluate_matlab_code` runs model-written code in your MATLAB session with your privileges, including anything MATLAB can reach on your filesystem and OS (`system`, `delete`, file writes). There is no sandbox between the agent and your machine; **review tool calls before approving them**, keep the agent's working folder scoped to a dedicated project directory, and do not point it at a session holding sensitive or irreplaceable data. One early user [demonstrated](https://www.mathworks.com/matlabcentral/discussions/ai/885947-experiments-with-claude-code-and-matlab-mcp-core-server) running OS-level shell commands straight through `evaluate_matlab_code`; treat "can run MATLAB" as "can run anything your account can." Treat a session-attached agent with the same caution as the desktop-driving agents in [section 2](#agents-that-drive-your-browser-or-desktop).

> [!NOTE]
> The agent reads **text** output and any files you save; it does not see figures rendered in the MATLAB UI. To let it "see" a plot, save the figure to an image file it can read (PNG or JPEG) – then it can open and reason about it. This is a small but frequent gotcha when you ask an agent to "check the plot."

### MATLAB as the client: calling LLMs and building agents in MATLAB

Use this when you do not want an assistant so much as to *program with* a model from inside MATLAB: classify free text, extract structured fields, build a domain chatbot, or run a model over many rows. The free **[Large Language Models (LLMs) with MATLAB](https://github.com/matlab-deep-learning/llms-with-matlab)** add-on (install from the Add-On Explorer, or clone the repo; requires R2024a+) is the core piece. It connects to the OpenAI Chat Completions API, Azure OpenAI, [**Ollama**](https://ollama.com/) (for open-weight models run locally or on Ollama's hosted cloud), and any other OpenAI-compatible API endpoint – so a self-hosted server or an institutional gateway such as a university-hosted LLM reached with an API key works too. It supports streaming, chat-history management, structured/JSON output, tool calling, and generating, editing, and describing images.

```matlab
chat = openAIChat("You are a terse assistant. American English.", ...
                  ModelName="gpt-4o");
[txt, response] = generate(chat, "Summarize heat-shock recovery in one line.");
```

Connecting to [a local Ollama model](https://docs.ollama.com/quickstart) keeps data on your machine and avoids per-token cost, which matters for sensitive data or classroom use. However, MATLAB Online currently only reaches OpenAI and Azure and not Ollama, and so Ollama use currently requires an installed MATLAB and not one that runs in the browser. The repository's [documentation](https://github.com/matlab-deep-learning/llms-with-matlab) is a good starting point, including its [Ollama guide](https://github.com/matlab-deep-learning/llms-with-matlab/blob/main/doc/Ollama.md).

#### Building MCP clients and agents in MATLAB

You can also have MATLAB act as an MCP *client*, connecting to MCP servers that are typically used for interfacing with AI agents. The **[MATLAB MCP HTTP Client](https://github.com/matlab-deep-learning/mcpHTTPClient)** (R2025a+, paired with LLMs with MATLAB 4.6.0+) lets MATLAB call external MCP tools with a single `callTool`, and so a model running inside MATLAB can decide for itself when to reach out to another MCP server. As the name says, it speaks **streamable HTTP** and so connects to remotely hosted servers, not to the local stdio servers that most desktop assistants register – including the MATLAB MCP Server itself. That makes it the basis for building your own agents (ReAct loops, tool chaining, human-in-the-loop) in plain MATLAB code. It is the mirror image of [MATLAB as the server](#matlab-as-the-server-the-matlab-mcp-server): there an outside agent drives MATLAB, whereas here MATLAB is the agent.

Furthermore, the newer **[MATLAB AI Agent SDK](https://github.com/matlab/matlab-ai-agent-sdk)** provides a more streamlined approach to **developing MATLAB-based AI agents:** `aisdk.AIAgent` ties together an LLM client (OpenAI, Ollama, or any OpenAI-compatible endpoint), your own functions exposed as tools with `aisdk.LLMTool`, and external MCP servers through `MCPTool`, handling message history, parallel tool calls, and subagents for you. Because a tool can operate on data already in your workspace, large arrays stay local rather than being shipped to the model. It is a **research preview** under active development, and so the API may not have yet stabilized, but it is the most direct path today to a MATLAB-native AI agent.

### Force multiplier for any editor: the bridge across the setups above

The MCP server upgrades any approach from sections 1–4 rather than replacing it. It lets the model run code in your real session and read the toolboxes you actually have, which removes the failure named at the top of this guide, where the model guesses at an array's shape, invents a function signature, or assumes a toolbox you do not own. How much that helps scales with how tightly it is wired in.

| Setup                                                                        | What the MATLAB MCP Server adds                                                                                                     |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Stand-alone chat ([claude.ai](https://claude.ai/), ChatGPT)                  | nothing automatic – you still paste; the server is for assistants that can call tools                                               |
| In-editor assistant (MATLAB Copilot; VS Code Chat)                           | register the server and the assistant can run code, check it, run tests, and enumerate toolboxes, not just read your open files     |
| Agentic tool, CLI or desktop (Claude Code in a terminal or its **Code** tab) | the server **plus** `shareMATLABSession()` lets the agent run against your *live* workspace mid-task, then edit `.m` files to match |

**A worked example.** Say Claude Code is running in a terminal and you ask it to fix a script that errors when indexing into a struct array of results. Without the bridge, the agent sees only the file: it guesses the field is `result` (it is actually `results`), edits the file, runs it with `matlab -batch` against an empty workspace, reads the error, and guesses again – a slow round-trip that fails outright if the data lives only in your interactive workspace and never as a file the agent can load. With a shared session, it instead calls `evaluate_matlab_code` to inspect the actual struct, reads the real field names and sizes, confirms the indexing, and makes the correct edit on the first pass. It can also call `detect_matlab_toolboxes` first and avoid proposing a function from a toolbox you have not licensed – or recover gracefully when it does: in MathWorks' [polynomial-fitting walkthrough](https://blogs.mathworks.com/matlab/2025/11/03/exploring-the-matlab-model-context-protocol-mcp-core-server-with-claude-desktop/), the agent's first attempt called `crossvalind` from a Bioinformatics Toolbox the author did not have, read the resulting error, and rewrote the script without it, all in one pass.

The wiring is just the pieces shown earlier in this section – register the server with your assistant, then `--setup-matlab` and `shareMATLABSession()` so the agent reaches the session you are actually working in. Those steps settle the statefulness catch in [sections 3](#3-coding-assistant-as-script-editor-edit-m-files-without-session-information) and [4](#4-editors-and-ides-that-simplify-use-of-coding-assistants-alongside-matlab) into a non-issue, and it is why the [quick start](#1-quick-start-llm-integrated-with-a-live-matlab-session) pairs Claude Code with the MCP server rather than running it blind.

## Cross-cutting cautions

- **Verify generated numerical code.** Plausible-looking MATLAB is not correct MATLAB. Vectorization that quietly changes broadcasting, the wrong dimension in a reduction, off-by-one indexing, and the meaning of a model object's properties all reward a careful read. Treat agent output as a fast first draft to react to, not an answer.
- **A correct tool result can still be reported wrong.** The MCP tools run real MATLAB, but the model's *summary* of what they returned is still an LLM guess. In MathWorks' own [walkthrough](https://blogs.mathworks.com/matlab/2025/11/03/exploring-the-matlab-model-context-protocol-mcp-core-server-with-claude-desktop/), `detect_matlab_toolboxes` returned an accurate list yet the model miscounted it (102 instead of 99). When a number matters, have the agent compute it *in* MATLAB rather than counting in prose.
- **Mind what the assistant can see, and do.** Live-session access lets the agent run arbitrary code with your privileges; UI-driving and desktop agents go further still. Decide consciously what each tool can reach – and keep its working folder scoped – before you wire it up.
- **Cost and keys.** Agentic and in-editor workflows bill against a subscription or API key and scale with context length and turns. Prefer cheaper models for routine tasks and reserve top-tier models for hard problems; for sensitive or high-volume work, a local Ollama model through [LLMs with MATLAB](#matlab-as-the-client-calling-llms-and-building-agents-in-matlab) avoids both cost and data exposure. Before paying for anything, check what you already have: an institutional LLM gateway, a self-hosted open-weight model, or a free student tier will drive a [bring-your-own-key](#bring-your-own-key-byok-provider-agnostic-agents) agent, and none of the session integration in this guide depends on which vendor supplies the model.
- **Licensing.** The MATLAB MCP Server is single-user and must not be shared by multiple users; check the [repository](https://github.com/matlab/matlab-mcp-server) terms for any deployment beyond your own machine.
- **Reproducibility.** If an LLM touched a result that goes into a paper, the pipeline should still run without it. Pin versions, script the calls, and keep a human-readable record of what the model did.

## Reproducibility and sandboxing

This is less an integration approach than the infrastructure that makes the others safe to lean on. Agents are most useful when their environment is pinned and their autonomy is bounded.

- **Pin your environment.** Record the MATLAB release and the toolbox versions a generated analysis depends on (the agent can capture them with `detect_matlab_toolboxes`/`ver`), and check them into the project so the analysis runs the same way next month and on a student's machine.
- **Scope and sandbox.** Keep the agent's working folder to a dedicated project directory, and for risky autonomy run MATLAB inside a container (MathWorks' [prebuilt MATLAB image](https://hub.docker.com/r/mathworks/matlab) on Docker Hub, or its [Dockerfiles](https://github.com/mathworks-ref-arch/matlab-dockerfile) if you would rather build your own) so an agentic CLI runs with less risk to your real filesystem. The tradeoff is that container isolation complicates attaching to the *interactive* desktop session of [section 5](#connecting-to-your-running-session-session-modes); a common pattern is to develop interactively with a shared session, then verify the final result inside a clean container.

## Use-case quick reference

| Goal                                                         | Reach for:                                                                                      |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| Ask one-off questions, paste an error                        | [A stand-alone chat](#copy-and-paste-stand-alone-chat)                                          |
| Stay in the desktop with docs-grounded help and completions  | [MATLAB Copilot](#the-matlab-desktop-and-matlab-copilot)                                        |
| Have an external agent operate MATLAB or a cloud session     | [A browser- or desktop-driving agent](#agents-that-drive-your-browser-or-desktop)               |
| Run MATLAB without a local install                           | [MATLAB Online](#where-matlab-runs-in-the-browser-cloud-options)                                |
| Hand off a multi-file task to an agent                       | [An agentic CLI](#3-coding-assistant-as-script-editor-edit-m-files-without-session-information) |
| Use a free, subsidized, or self-hosted model                 | [A Bring-Your-Own-Key agent](#bring-your-own-key-byok-provider-agnostic-agents)                 |
| Let an agent run code in your real session and see toolboxes | [The MATLAB MCP Server](#matlab-as-the-server-the-matlab-mcp-server)                            |
| Call an LLM *from inside* your MATLAB code                   | [LLMs with MATLAB](#matlab-as-the-client-calling-llms-and-building-agents-in-matlab)            |
| Pin versions or sandbox an autonomous agent                  | [Docker and version pinning](#reproducibility-and-sandboxing)                                   |

## Selected references

- MATLAB MCP Server: <https://github.com/matlab/matlab-mcp-server>
- Exploring the MATLAB MCP Core Server with Claude Desktop (MathWorks blog): <https://blogs.mathworks.com/matlab/2025/11/03/exploring-the-matlab-model-context-protocol-mcp-core-server-with-claude-desktop/>
- Experiments with Claude Code and the MATLAB MCP Core Server (MATLAB Central discussion): <https://www.mathworks.com/matlabcentral/discussions/ai/885947-experiments-with-claude-code-and-matlab-mcp-core-server>
- MATLAB MCP Core Server Update: Bringing Your Coding Guidelines Directly to AI (MathWorks blog): <https://blogs.mathworks.com/deep-learning/2025/12/11/matlab-mcp-server-update-bringing-your-coding-guidelines-directly-to-ai/>
- MATLAB Agentic Toolkit: <https://github.com/matlab/matlab-agentic-toolkit>
- Simulink Agentic Toolkit: <https://github.com/matlab/simulink-agentic-toolkit>
- Guide to Agentic AI with MATLAB and Simulink (MathWorks): <https://www.mathworks.com/campaigns/offers/next/agentic-ai-with-matlab.html>
- MATLAB MCP HTTP Client (MATLAB as an MCP client): <https://github.com/matlab-deep-learning/mcpHTTPClient>
- MATLAB AI Agent SDK (build agents in MATLAB; research preview): <https://github.com/matlab/matlab-ai-agent-sdk>
- MATLAB extension for Visual Studio Code: <https://github.com/mathworks/MATLAB-extension-for-vscode>
- Using MATLAB in Visual Studio Code: <https://www.mathworks.com/products/matlab-with-vs-code.html>
- MATLAB Copilot / Generative AI with MATLAB: <https://www.mathworks.com/products/matlab/generative-ai.html>
- MATLAB Copilot FAQ (licensing and eligibility): <https://www.mathworks.com/products/matlab-copilot/faq.html>
- AI language models in VS Code (bring your own key): <https://code.visualstudio.com/docs/agent-customization/language-models>
- Large Language Models (LLMs) with MATLAB: <https://github.com/matlab-deep-learning/llms-with-matlab>
- MATLAB Online: <https://matlab.mathworks.com/>
- Model Context Protocol: <https://modelcontextprotocol.io/>
- Claude Code: <https://www.anthropic.com/claude-code>
- Codex CLI: <https://developers.openai.com/codex/cli/>
- Antigravity (CLI and IDE; successor to Gemini CLI): <https://antigravity.google/>
- Transitioning Gemini CLI to Antigravity CLI (Google Developers Blog): <https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/>
- Companion guide: [Using LLMs with R](using-llms-with-r.md)

---

*MIT License. Copyright © 2026 Theodore P. Pavlic. Shared as a starting point; verify against current documentation, because this tooling changes quickly.*
