# Local AI Coding Assistant Manual

**(Free, Offline, Open-Source Research Setup)**
For: *Python • C++ • Bash • LaTeX • Markdown*
Works on: *VS Code, VSCodium, and Neovim*

---

## ⚙️ 1. Overview

| What You Get                    | Description                                    |
| ------------------------------- | ---------------------------------------------- |
| 💬 **AI Chat Inside Editor**    | Talk to your AI about any code or repo issue   |
| 🧩 **Understands Full Repo**    | Reads and reasons about your local Git project |
| 🧠 **Multi-language Support**   | Python, C++, Bash, LaTeX, Markdown             |
| 🔒 **Fully Local & Private**    | No cloud uploads, no telemetry                 |
| 🆓 **Free & Open-Source**       | Everything runs offline via open models        |
| 🧰 **Easy to Install & Remove** | Clean setup and teardown                       |
| 🐚 **Fish Shell Integration**   | Manage AI models with short commands           |

---

## 🔹 2. Install Ollama

Ollama is your local model server (think “offline ChatGPT engine”).

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Check if it’s working:

```bash
ollama --version
```

Start the service:

```bash
sudo systemctl start ollama
```

### Troubleshooting: Black Screen on Suspend/Resume (NVIDIA Hybrid Graphics Laptops)
If you encounter a black screen after waking from sleep (common on Ubuntu with Intel + NVIDIA setups like GeForce 940MX), Ollama's CUDA GPU usage can interfere with power management.

**Quick Test (Non-Destructive):**
```bash
sudo systemctl stop ollama
```
Test suspend (close lid or `systemctl suspend`). If it works, proceed to fix.

**Permanent Fix:**
Mask NVIDIA's faulty suspend services (kernel handles GPU power instead):
```bash
sudo systemctl mask nvidia-suspend.service nvidia-hibernate.service nvidia-resume.service
sudo reboot
```
- Restart Ollama: `sudo systemctl start ollama`.
- Test suspend again—should resolve without losing GPU acceleration.
- Undo if needed: `sudo systemctl unmask nvidia-suspend.service nvidia-hibernate.service nvidia-resume.service && sudo reboot`.

This affects only NVIDIA users; Intel-only setups are unaffected.

---

## 🔹 3. Install the Best Free Model (per 2025 benchmarks)

| Model                    | Strength                    | HumanEval % | RAM     | Notes                   |
| ------------------------ | --------------------------- | ----------- | ------- | ----------------------- |
| 🥇 **Qwen 2.5 Coder 7B** | Research coding, multi-lang | **88.4%**   | ~4–8 GB | Top choice (your setup) |
| 🥈 DeepSeek-Coder 6.7B   | Structural reasoning        | 78.6%       | ~6 GB   | Good backup             |
| 🥉 Mistral 7B            | General chat + coding       | 65%         | ~4 GB   | Lightweight fallback    |

**Pull the main model:**

```bash
ollama pull qwen2.5-coder:7b
```

Optionally:

```bash
ollama pull deepseek-coder:6.7b
```

List models:

```bash
ollama list
```

Test directly:

```bash
ollama run qwen2.5-coder:7b
```

---

## 🔹 4. Add Fish Helper Script: `ollama-fish`

To manage Ollama easily in your Fish shell.

Create file:

```bash
mkdir -p ~/.config/fish/functions
nvim ~/.config/fish/functions/ollama-fish.fish
```

Paste this:

```fish
function ollama-fish --description "Quick Ollama manager for research coding"
    switch $argv[1]
        case pull
            set model $argv[2]
            if test -z "$model"
                echo "Usage: ollama-fish pull <model> (e.g., qwen2.5-coder:7b)"
                return 1
            end
            echo "🚀 Pulling $model (~4-5GB for 7B models)..."
            ollama pull $model
            echo "✅ Done! Run 'ollama-fish list' to check."

        case start
            echo "🟢 Starting Ollama service..."
            sudo systemctl start ollama
            ollama serve &
            echo "✅ Running on localhost:11434."

        case stop
            echo "🔴 Stopping Ollama..."
            sudo systemctl stop ollama
            pkill ollama
            echo "✅ Stopped."

        case list
            echo "📋 Installed models:"
            ollama list | grep -v '^NAME'

        case test
            set model $argv[2]; or set model qwen2.5-coder:7b
            echo "🧪 Testing $model..."
            curl -s http://localhost:11434/api/generate -d "{
              'model': '$model',
              'prompt': 'Write a Python snippet that loads a CSV, analyzes columns, and plots results.',
              'stream': false
            }" | jq -r '.response' | head -c 300
            echo "... (full results visible in Continue chat)"

        case rm
            set model $argv[2]
            if test -z "$model"
                echo "Usage: ollama-fish rm <model>"
                return 1
            end
            echo "🗑️ Removing $model..."
            ollama rm $model
            echo "✅ Removed."

        case '*'
            echo "Usage: ollama-fish (pull|start|stop|list|test|rm)"
    end
end
```

Reload Fish:

```bash
source ~/.config/fish/functions/ollama-fish.fish
```

Now you can manage Ollama easily:

```bash
ollama-fish pull qwen2.5-coder:7b
ollama-fish start
ollama-fish list
ollama-fish test
```

---

## 🔹 5. Install Continue.dev in VS Code / VSCodium

### VS Code:

* `Ctrl + Shift + X` → search **Continue** → Install
  *(Official extension: `Continue - Open Source Autopilot`)*

### VSCodium:

```bash
codium --install-extension Continue.continue
```

You’ll now see a **💬 Continue** icon in the sidebar.

---

## 🔹 6. Configure Continue to Use Qwen (Local Model)

Open config:

```
Ctrl + Shift + P → “Continue: Open Settings” → Agents → Configure Local Agent
```
Or directly open: `~.continue/config.yaml`
Paste this:

```yaml
name: Local Agent
version: 1.0.0
schema: v1
models:
  - name: Qwen 2.5 Coder 7B
    provider: ollama
    model: qwen2.5-coder:7b
    apiBase: http://localhost:11434
    roles:
      - chat
      - edit
      - apply
      - autocomplete
    capabilities:
      - tool_use  # For agent-like research workflows if expanded later
    defaultCompletionOptions:
      temperature: 0.2  # Precise code gen/refactors
      maxTokens: 2048
      stop: ["\n\n", "###"]
    autocompleteOptions:
      debounceDelay: 300
      maxPromptTokens: 1024
      onlyMyCode: true  # Repo-focused
  - name: Nomic Embed
    provider: ollama
    model: nomic-embed-text:latest
    roles:
      - embed
```

Save → reload window:

```
Ctrl + Shift + P → “Developer: Reload Window”
```

---

## 🔹 7. (Optional) Add Research Slash Commands

Add these to the end of your same `config.yaml`:

```yaml
prompts:
  - name: research-refactor
    description: Refactor research project structure
    prompt: |
      @Files Suggest a modular layout separating Python analysis, Bash preprocessing, C++ compute, and LaTeX reporting. Output refactored folder structure and rationale.
  - name: bug-hunt
    description: Find and fix bugs in multi-lang code
    prompt: |
      @code Analyze the bug across Python/Bash/C++. Provide minimal fixes and test suggestions.
  - name: pipeline-optimize
    description: Optimize data pipeline
    prompt: |
      @Files Review scripts for efficiency. Suggest parallelization, vectorization, or better I/O handling.
  - name: doc-gen
    description: Generate LaTeX/MD docs from code
    prompt: |
      @Files Extract methods and results, format them into a LaTeX/Markdown report.

```

Now in Continue chat, type `/research-refactor` or `/pipeline-optimize`.

---

## 🔹 8. Example Uses

In Continue chat:

```
@workspace explain the overall architecture of my project
@workspace improve the C++ build and Bash automation
@workspace check Python scripts for redundant loops
@workspace write a LaTeX section describing data analysis method
```

Or highlight code → `Ctrl + L` → “Refactor with Continue”.

---

## 🔹 9. Remove Everything

**Remove Ollama + models**

```bash
sudo systemctl stop ollama
sudo systemctl disable ollama
sudo rm -rf /usr/local/bin/ollama ~/.ollama
```

**Unmask NVIDIA Services (If You Applied the Fix)**
```bash
sudo systemctl unmask nvidia-suspend.service nvidia-hibernate.service nvidia-resume.service
sudo reboot
```

**Remove Continue**

```bash
code --uninstall-extension Continue.continue
# or
codium --uninstall-extension Continue.continue
```

**Remove Fish helper**

```bash
rm ~/.config/fish/functions/ollama-fish.fish
```

---

## 🔹 10. Summary

| Feature                            | Supported | Notes                                  |
| ---------------------------------- | --------- | -------------------------------------- |
| Offline + private                  | ✅         | No data leaves your machine            |
| Repo-wide analysis                 | ✅         | Reads your entire local repo           |
| Multi-language (Py/C++/Bash/LaTeX) | ✅         | Fully supported                        |
| Free + open source                 | ✅         | Ollama + Continue                      |
| Editor integration                 | ✅         | Works with VS Code / VSCodium / Neovim |
| RAM required                       | ~8–10 GB  | Fits your 24 GB easily                 |
| Disk space                         | ~4–5 GB   | For Qwen 2.5 Coder 7B                  |

---

## 🧩 Everything You Want — 100% Open Source

✅ Local AI (Ollama)
✅ Local chat IDE integration (Continue)
✅ Offline large model (Qwen 2.5 Coder 7B)
✅ Fish shell helper (for quick use)
✅ No file upload needed
✅ Full project analysis
✅ Easy uninstall anytime
