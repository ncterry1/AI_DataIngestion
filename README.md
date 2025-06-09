Below is the **fully detailed, annotated lab-setup guide**. Each section is expanded with “What,” “Why,” and “How” notes—much like Section 5—so you not only see the commands but also understand the rationale behind them. Follow the sections in order from top to bottom.

---

## 1. Host Machine Preparation

Before spinning up any virtual machine, ensure your laptop can handle the load. Think of this as making sure your car has enough fuel, correct tires, and working brakes before a long drive.

### 1.1. Verify Hardware & Software Requirements

1. **RAM (Memory)**

   * **Minimum**: 16 GB of RAM.
   * **Recommended**: 24 GB or more, especially if you plan to run CPU-based LLM inference (which can consume significant memory) alongside Docker/Neo4j.
   * **Why?**
     ‒ Ubuntu alone can consume \~1–2 GB.
     ‒ Neo4j (in Docker) can reserve 2–4 GB for heap + page cache.
     ‒ A 7 B-parameter LLaMA model on CPU may use \~12–14 GB.
     ‒ VS Code, other background processes, and buffer for OS: \~2–4 GB.
     Total can approach or exceed 16 GB if everything runs simultaneously.

2. **CPU and Virtualization**

   * **Requirement**: A CPU with virtualization extensions (Intel VT-x or AMD-V).
   * **How to Check**:

     * On Windows host: Open **Task Manager → Performance → CPU**. Look for “Virtualization: Enabled.”
     * On Linux host:

       ```bash
       egrep -c '(vmx|svm)' /proc/cpuinfo
       ```

       If the result is ≥ 1, your CPU supports VT-x or AMD-V.
   * **What Happens If Off?**
     Virtualization disabled in BIOS/UEFI → VM cannot start. You’ll get an error like “VT-x is disabled in the BIOS.”

3. **Disk Space**

   * **Minimum Free Space**: 200 GB.
   * **Why?**
     ‒ Ubuntu VM’s thin-provisioned virtual disk: up to 100 GB (but initially takes only what’s used).
     ‒ Docker images (Neo4j can be \~1 GB), container data, and cached LLM weights (\~10–20 GB) in `~/.cache/huggingface`.
     ‒ Snapshots: Taking snapshots of a 100 GB VM can double disk usage temporarily.
     Total reserves \~200 GB to avoid “disk full” issues.

4. **VMware Workstation Pro (or Fusion Pro)**

   * **Install/Update**: If not installed, download and install the latest VMware Workstation Pro (Windows/Linux) or VMware Fusion Pro (macOS).
   * **Why VMware Pro?**
     ‒ Supports advanced features like snapshots, easy hardware customization, and (on some systems) limited GPU passthrough.
   * **Check for Updates**:
     In VMware: **Help → Software Updates**, then install any available patches.

5. **Create a Host Directory for VMs**

   * **Path Example**:

     * Windows: `C:\VMs\AI_Lab`
     * macOS/Linux: `/Users/youruser/VMs/AI_Lab` or `/home/youruser/VMs/AI_Lab`
   * **Why?**
     ‒ Keeps all virtual machine files (disks, config) in one place.
     ‒ Facilitates backups and snapshots.
     ‒ Prevents scattering `.vmdk`, `.vmx` files across your host’s user folder.

6. **Decide on Network Mode**

   * **Use NAT** (Network Address Translation).
     ‒ VM obtains a private IP (commonly 192.168.x.x) from VMware’s DHCP.
     ‒ Outbound internet is routed through the host’s IP.
     ‒ Ports forwarded from VM to host need explicit mapping (e.g., Docker’s port 7474 on VM → port 7474 on host).
   * **Why NAT?**
     ‒ Isolates the VM from your local LAN. A service running on VM (Neo4j) is reachable only if you map ports or use SSH forward.
     ‒ Simplifies network setup—no need to manage IP conflicts on the LAN.
   * **Alternative**: **Bridged** networking. VM becomes a peer on your LAN (gets an IP from your router). Useful if you want other devices on your network to reach your VM by IP, but this can expose open ports to the LAN by default.

---

## 2. Create a Single Ubuntu VM (Inference + Neo4j)

We’ll host **both** LLM inference and Docker-hosted Neo4j on **one** Ubuntu VM. This minimizes resource overhead compared to separate VMs.

### 2.1. Launch VMware and Create a New Virtual Machine

1. **Open VMware Workstation Pro (or Fusion Pro)**.

2. **File → New Virtual Machine → Typical (Recommended)**

   * *Why “Typical”?* It walks you through the essential options without overwhelming details.

3. **Select the Installer**

   * Choose the **Ubuntu 22.04 LTS ISO** you downloaded.
   * If you haven’t downloaded it: go to [ubuntu.com/download/desktop](https://ubuntu.com/download/desktop) and grab the latest **Ubuntu 22.04 LTS** for x86\_64.
   * Click **Next**.

4. **Guest Operating System**

   * VMware should detect: **Linux → Ubuntu 64-bit**.
   * Click **Next**.

5. **Name and Location**

   * **Name**: `Ubuntu_AI_Lab` (or something descriptive).
   * **Location**: Browse to your host directory, e.g., `C:\VMs\AI_Lab\Ubuntu_AI_Lab`.
   * *Why specify location?* Keeps all related VM files in one folder.
   * Click **Next**.

6. **Disk Capacity**

   * **Specify disk size**: 100 GB.
   * **Store virtual disk as a single file**: Checked.
     ‒ Thin provisioning by default: VMware will not pre-allocate all 100 GB; it grows as we write data (but never beyond 100 GB).
   * Click **Next**.

7. **Customize Hardware** (Important)

   * Click **Customize Hardware** before finishing.

     1. **Memory**: Set to **12 GB** (12,288 MB).

        * If your host has 24 GB or more, consider **16 GB** to give extra headroom.
        * Click into the memory field and type `12288` or use the slider.
     2. **Processors**:
        ‒ **Number of processor cores**: 4 vCPUs.
        ‒ If your host has 8 or more physical threads, you can allocate more, but 4 is a decent balance.
     3. **Network Adapter**:
        ‒ Ensure it’s set to **NAT** (so the VM sits behind the host’s IP).
     4. **Display**:
        ‒ **Accelerate 3D graphics**: Checked.
        • This installs VMware’s SVGA II driver in Ubuntu for smoother UI and any light GPU-accelerated tasks (video playback, desktop compositing).
        • Note: This is not the same as CUDA/GPU passthrough—see Section 8 for GPU.
     5. **New CD/DVD (IDE)**:
        ‒ Make sure it’s set to point to the Ubuntu 22.04 ISO file.
   * Click **Close** to save hardware settings.

8. **Finish**

   * Review summary: Ubuntu 22.04 LTS, 4 vCPUs, 12 GB RAM, 100 GB disk (thin), NAT.
   * Click **Finish**. VMware creates the `.vmdk` (virtual disk) and `.vmx` (configuration) files under your chosen location.

### 2.2. Install Ubuntu on the VM

1. **Power On the VM**

  NCT - note that when you get here it give you the option to download updates. Don't do this yet, just skip and go through the extended installation steps while in Ubuntu. Do the updates through terminal later.
   * Select `Ubuntu_AI_Lab` and click **Power On**.
   * The VM will boot from the ISO and show the Ubuntu installer.

2. **Follow Ubuntu Installer Steps**

   * **Language**: Select **English** (or your preference). Click **Install Ubuntu**.
   * **Keyboard Layout**: Usually **English (US)**, but choose yours if different. Click **Continue**.
   * **Updates and Other Software**:
     ‒ Select **Normal installation** (includes utilities, browsers).
     ‒ Check **Install third-party software** (so you get hardware drivers, MP3 codecs).
     ‒ Optional: uncheck **Download updates while installing** if your internet is slow; you can run `sudo apt update` later.
   * **Installation Type**:
     ‒ Choose **Erase disk and install Ubuntu** (because this VM’s virtual disk has nothing else on it).
   * **Time Zone**:
     ‒ Select your region (e.g., **America → New York**). Click **Continue**.
   * **User Setup**:
     ‒ **Your name**: e.g., `Lab User`.
     ‒ **Your computer’s name**: e.g., `ubuntu-ai-lab`.
     ‒ **Username**: `labuser`.
     ‒ **Password**: Choose a strong password.
     ‒ Uncheck “Log in automatically” if you want to enter a password each time.
   * Click **Continue**. Ubuntu will begin installing—this may take 5–10 minutes.

3. **Reboot and Remove ISO**

   * When installation finishes, click **Restart Now**.
   * Remove the ISO from the virtual drive if VMware asks, or let VMware eject it automatically.
   * The VM reboots into a fresh Ubuntu desktop.

### 2.3. Post-Install Configuration

1. **Log In**

   * Use the credentials you set (e.g., `labuser` / your password).
   * You’ll see Ubuntu’s GNOME desktop.

2. **Open a Terminal** (Ctrl + Alt + T).

3. **Update & Upgrade Packages**

   ```bash - NCT Done
   sudo apt update && sudo apt upgrade -y
   ```

   * **Why?**
     ‒ Ensures you have the latest security patches and bug fixes.
     ‒ Might update the kernel, drivers, and default packages.

4. **Install Common Tools**

   ```bash - NCT done
   sudo apt install -y build-essential git curl wget python3 python3-venv python3-pip
   ```

   * **`build-essential`**: compilers (`gcc`, `g++`), `make`, and related tools—helpful if you compile anything from source.
   * **`git`**: version control for pulling code.
   * **`curl`/`wget`**: utilities for downloading files from the command line.
   * **`python3`**, **`python3-venv`**, **`python3-pip`**: Python 3 environment, essential for running LLM scripts.

5. **Reboot (Optional but Suggested)**

   ```bash - NCT done
   sudo reboot
   ```

   * Ensures any kernel or driver updates from `apt upgrade` fully apply.

6. **Verify Virtualization Extensions Inside the VM**

   ```bash - NCT done. Worked but nothing new appeared
   grep -E '(vmx|svm)' /proc/cpuinfo
   ```

   * If you see flags `vmx` (Intel) or `svm` (AMD), your VM is running on a virtualized CPU that itself supports nested virtualization.
   * **Why check?**
     ‒ If you ever need to run nested VMs inside this VM (unlikely)—just good to verify.

---

## 3. Install Docker & Docker Compose on the Primary Ubuntu VM

All Docker-related actions occur **inside this single Ubuntu VM** (no separate VM for Neo4j). Docker will host Neo4j in a container.

### 3.1. Install Docker Engine

1. **Run Docker’s Official Convenience Script**

   ```bash - NCT done
   curl -fsSL https://get.docker.com | sudo sh
   ```

   * **What This Does**:

     1. Adds Docker’s GPG key.
     2. Creates `/etc/apt/sources.list.d/docker.list` pointing to Docker’s apt repo.
     3. Installs `docker-ce` (Community Edition), `docker-ce-cli`, and `containerd.io`.
   * **Why Use the Script?**
     ‒ Simplifies the steps. If you prefer manual steps, see [docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu/).
   * **Note**:
     ‒ If you want to pin a specific Docker version, you’d manually add the repo, run `sudo apt update`, then `sudo apt install docker-ce=<version> docker-ce-cli=<version> containerd.io`.

2. **Add Your User to the Docker Group**

   ```bash - NCT done
   sudo usermod -aG docker $USER
   ```

   * **Why?**
     ‒ By default, only root or users in the `docker` group can run `docker` commands.
     ‒ Adding `labuser` to `docker` lets you run `docker ps`, `docker run`, etc., without `sudo`.
   * **Important**:
     ‒ To apply this change, **log out and back in**, or run:

     ```bash - NCT done
     newgrp docker
     ```

     ‒ Confirm group membership with:

     ```bash - NCT done, docker listed
     groups
     ```

     You should see `docker` listed.

3. **Verify Docker Installation**

   ```bash - NCT done
   docker version
   docker info
   ```

   * **Expected Output**:
     ‒ `Client: Docker Engine - Community` with version details.
     ‒ `Server: Docker Engine - Community` with details.
   * **If You See Errors**:
     ‒ Check if `docker` daemon is running:

     ```bash - NCT checked
     sudo systemctl status docker
     ```

     ‒ If stopped, start it:

     ```bash - NCT ok, running
     sudo systemctl start docker
     ```

     ‒ If still failing, check `/var/log/syslog` or `journalctl -u docker` for clues.

### 3.2. Install Docker Compose Plugin

> **Why Docker Compose?**
> Instead of manually writing `docker run` commands to build the Neo4j container, we use a `docker-compose.yml` file to declare services, ports, volumes, and environment variables in one place. This ensures reproducibility and clarity.

1. **Install via APT**

   ```bash - NCT done
   sudo apt install -y docker-compose-plugin
   ```

   * **Result**:
     ‒ The command `docker compose` (note the space) becomes available.
     ‒ You no longer use the older separate `docker-compose` binary; you run `docker compose up`, `docker compose down`, etc.

2. **Verify**

   ```bash - NCT done
   docker compose version
   ```

   * **Expected**: A version string like `Docker Compose version v2.x.x`
   * **If Not Found**:
     ‒ Double-check the package installed.
     ‒ Ensure you ran `sudo apt update` recently.

---

## 4. Deploy Neo4j via Docker Compose on the Ubuntu VM

We’ll create a folder (`~/ai_graph_lab`) containing a `docker-compose.yml` file to run Neo4j Community Edition in a container.

### 4.1. Create a Directory for Neo4j Configuration & Data

1. **Make the Folder**

   ```bash - NCT done
   mkdir -p ~/ai_graph_lab
   cd ~/ai_graph_lab
   ```

   * **Why `~/ai_graph_lab`?**
     ‒ Keeps the Docker Compose file and persistent data organized in one place.
     ‒ Under this directory, we’ll have:

     * `docker-compose.yml`
     * `neo4j_data/` (volume mount for Neo4j database files)

2. **Verify Permissions**

   ```bash - NCT done
   ls -ld .
   ```

   * Ensure `labuser` owns the folder and can read/write. If you see root ownership, run:

     ```bash - NCT done, remember, labuser=ncterry
     sudo chown -R labuser:labuser ~/ai_graph_lab
     ```

### 4.2. Write `docker-compose.yml`

1. **Open an Editor**

   ```bash - NCT done
   nano docker-compose.yml
   ```

2. **Paste the Following Configuration** - NCT done

   ```yaml

   services:
     neo4j:
       image: neo4j:4.4.25-community
       container_name: neo4j-lab
       # Port mappings: HostPort:ContainerPort
       ports:
         - "7474:7474"    # HTTP UI (Neo4j Browser)
         - "7687:7687"    # Bolt protocol for drivers
       environment:
         # Initial username/password. Neo4j forces a password change on first login.
         NEO4J_AUTH: "neo4j/StrongPassword123"
         # Listen on all network interfaces (not just localhost)
         NEO4J_dbms_default__listen__address: "0.0.0.0"
       volumes:
         # Persist data (graph files, logs) to the host
         - ./neo4j_data:/data
   ```

   **Line-by-Line Clarification**:

   * **`services:`**: We declare a service named `neo4j`.
   * **`image: neo4j:4.5`**: Pulls the official Neo4j 4.5 Community Edition image from Docker Hub.
     ‒ You could use `neo4j:5.9-community` for Neo4j 5.x.
   * **`container_name: neo4j-lab`**: Sets a custom container name for easier management (`docker ps` will show `neo4j-lab` instead of random).
   * **`ports:`**: Maps host ports to container ports:
     ‒ `7474:7474` allows us to open `http://localhost:7474` on the VM’s browser (or host machine’s browser) and reach Neo4j Browser.
     ‒ `7687:7687` lets Python scripts (or external apps) connect via Bolt protocol at `bolt://localhost:7687`.
   * **`environment:`**: Sets environment variables inside the container.
     ‒ `NEO4J_AUTH: "neo4j/StrongPassword123"` establishes user `neo4j` with password `StrongPassword123`. Neo4j will force you to change it on first login.
     ‒ `NEO4J_dbms_default__listen__address: "0.0.0.0"` instructs Neo4j to accept connections on any network interface inside the container.
   * **`volumes:`**:
     ‒ `./neo4j_data:/data` mounts the host folder `~/ai_graph_lab/neo4j_data` to the container’s `/data` directory, where Neo4j stores its database files, logs, and configuration.
     ‒ **Why Persist Data?** If you stop and remove the container, your graph is not lost—the data remains in `neo4j_data`.

3. **Save and Exit**

   * In `nano`: press **Ctrl + O** (write out), then **Enter**, then **Ctrl + X** (exit).

### 4.3. Launch the Neo4j Container

1. **Bring Up the Container**

   ```bash - NCT Done, in the yml, note I had to change the 'image: neo4j:4.4.25-community', it was another before that would not compose
   mkdir -p ~/ai_graph_lab/neo4j_data
   cd ~/ai_graph_lab
   docker compose up -d
   ```

   * **`docker compose up -d`**:
     ‒ Downloads (if not already pulled) `neo4j:4.5` image.
     ‒ Creates and starts a container named `neo4j-lab` in detached (`-d`) mode.
   * **First-Time Download**:
     ‒ The image is \~500–600 MB. The download may take a minute or two depending on your internet.

2. **Verify the Container Is Running**

   ```bash - NCT Done
   docker ps
   ```

   * **Expected Output**:

     ```
     CONTAINER ID   IMAGE        COMMAND                  ...   PORTS                                  NAMES
     abcdef123456   neo4j:4.5    "/sbin/tini -- /bin/…"   ...   0.0.0.0:7474->7474/tcp, 0.0.0.0:7687->7687/tcp   neo4j-lab
     ```
   * If `STATUS` is `Up (healthy)` (or similar), Neo4j is running. If you see `Restarting` or `Exited`, inspect logs:

     ```bash - NCT it was up and healthy.Did not run this
     docker logs neo4j-lab
     ```

     ‒ Look for errors like “out of memory” or permission issues.

3. **Access Neo4j Browser in Ubuntu VM** - NCT - done, connected

   * Open Firefox (or any browser) inside the Ubuntu VM.
   * Navigate to:

     ```
     http://localhost:7474
     ```
   * Neo4j Browser login screen appears.
   * **Login**:
     ‒ **Username**: `neo4j`
     ‒ **Password**: `StrongPassword123`
   * Neo4j forces you to change the password on first login. Choose something memorable (e.g., `LabNeo4jP@ss`).

4. **Verify Data Persistence Folder**

   * While Ubuntu VM’s terminal is open, run:

     ```bash - NCT done, confirmed
     ls ~/ai_graph_lab/neo4j_data
     ```
   * You should see subdirectories like `databases`, `logical_logs`, `neo4j.conf`, `plugins`, etc. That confirms the volume mapping is working.

---

## 5. Set Up the LLM/Inference Environment on Ubuntu

This section provides in-depth guidance on installing Python, creating a virtual environment, choosing an LLM, testing CPU inference, and (optionally) exploring a local chat UI.

### 5.0. “What Is an LLM, and Which One Are We Using?”

1. **What Is a Large Language Model (LLM)?**

   * An LLM is a neural network trained on massive text corpora (billions to trillions of tokens).
   * It can generate human-like text, answer questions, summarize, translate, etc.
   * Architecturally, many LLMs use the **Transformer** model (Vaswani et al., 2017) with layers of self-attention.

2. **Our LLM of Choice**

   * **Model**: `"huggyllama/llama-7b-hf-cpu"` (7 billion parameters).
   * **Why LLaMA?**
     ‒ Meta’s LLaMA weights are open (subject to license) on Hugging Face.
     ‒ A CPU-quantized variant allows inference on machines without GPUs—great for testing on a laptop.
     ‒ If you later enable GPU passthrough, you can switch to a **4-bit quantized LLaMA-2 7B** or larger.

3. **Local vs. Hosted API**

   * We perform **local** inference by downloading model weights to the VM and running them in Python.
   * **Advantages**:
     ‒ No external API calls or latency.
     ‒ Full control over model version.
     ‒ No usage costs or rate limits.
   * **Trade-off**:
     ‒ Requires significant memory/compute resources.
     ‒ CPU inference is slower than GPU or cloud-based inference.

### 5.1. Install Python & Create a Virtual Environment

#### 5.1.1. Ensure Python 3 Is Installed

1. **Open a Terminal** (Ctrl + Alt + T).

2. **Install Python 3 and Virtualenv**

   ```bash - NCT done
   sudo apt update
   sudo apt install -y python3 python3-venv python3-pip
   ```

   * **`python3-venv`**: provides the `python3 -m venv` module to create isolated environments.
   * **`python3-pip`**: ensures you can install packages from PyPI.

3. **Verify Python Version**

   ```bash - NCT done, Python 3.12.3, pip 24.0
   python3 --version
   pip3 --version
   ```

   * Expect something like `Python 3.10.4` and `pip 23.0.1` (versions may vary).

#### 5.1.2. Create & Activate a Virtual Environment

1. **Navigate to Home Directory**

   ```bash - NCT done
   cd ~
   ```

2. **Create Virtualenv Named `venv-llm`**

   ```bash - NCT done
   python3 -m venv venv-llm
   ```

   * **Result**: A new folder `~/venv-llm` containing an isolated Python interpreter and standard library.
   * **Why Virtualenv?**
     ‒ Prevents conflicts between system Python packages and project-specific ones (e.g., `torch`, `transformers`).
     ‒ Makes it easy to reproduce the environment on another machine.

3. **Activate the Virtualenv**

   ```bash - NCT done
   source ~/venv-llm/bin/activate
   ```

   * Your prompt changes to `(venv-llm) labuser@ubuntu:~$`.
   * While this is active, `pip install` goes into `~/venv-llm/lib/python3.10/site-packages`.

4. **Upgrade `pip`**

   ```bash - NCT done
   pip install --upgrade pip
   ```

   * Ensures you have the latest package installer for bug fixes and compatibility.

### 5.2. Install Core ML Dependencies

1. **Install PyTorch, Transformers, and SentencePiece**

   ```bash - NCT done
   pip install torch torchvision transformers sentencepiece
   ```

   * **`torch`**: The core PyTorch library—provides tensor operations, GPU support (if available), autograd, and model serialization.
   * **`torchvision`**: Typically used for computer vision; we install it because some `transformers` models rely on it for utilities.
   * **`transformers`**: Hugging Face’s library to load/download pre-trained LLMs (tokenizers, model weights, inference utilities).
   * **`sentencepiece`**: A library by Google to tokenize text into subword units. Many LLaMA-style models rely on SentencePiece.

2. **Clarifications on CPU vs. GPU Builds**

   * By default, `pip install torch` on Ubuntu will fetch a **CPU-only** build (unless you specify an index URL for CUDA).
   * Later (Section 8), if GPU passthrough is functional, you’ll uninstall `torch` and reinstall a CUDA-enabled wheel.

3. **Confirm Installation**

   ```bash - NCT done
   python3 -c "import torch, transformers, sentencepiece; print('Packages loaded successfully')"
   ```

   * No output errors means everything installed correctly.

### 5.3. Test a CPU-Only LLaMA Model

1. **Write a Quick Test Script**

   ```bash
   mkdir -p ~/lab_data
   nano ~/lab_data/test_llm.py
   ```

2. **Paste the Following Code into `test_llm.py`:**

   ```python
   # test_llm.py
   from transformers import AutoModelForCausalLM, AutoTokenizer

   def main():
       model_name = "huggyllama/llama-7b-hf-cpu"
       print(f"Loading tokenizer for {model_name}...")
       tokenizer = AutoTokenizer.from_pretrained(model_name)

       print(f"Loading model {model_name} (CPU-only inference)...")
       model = AutoModelForCausalLM.from_pretrained(model_name)

       prompt = "In a cozy room, two friends talk about security."
       print("Tokenizing prompt...")
       inputs = tokenizer(prompt, return_tensors="pt")

       print("Generating up to 30 new tokens...")
       outputs = model.generate(inputs.input_ids, max_new_tokens=30)

       text = tokenizer.decode(outputs[0], skip_special_tokens=True)
       print("\n===== GENERATED OUTPUT =====\n")
       print(text)
       print("\n============================\n")

   if __name__ == "__main__":
       main()
   ```

   **Clarifications**:

   * **`AutoTokenizer.from_pretrained(...)`** downloads or reads the tokenizer configuration and vocabulary.
   * **`AutoModelForCausalLM.from_pretrained(...)`** downloads or loads the model weights (\~14 GB).
   * **`max_new_tokens=30`** means generate 30 tokens beyond the prompt.
   * If you run out of memory, use a smaller model (e.g., `"huggyllama/llama-3b-hf-cpu"` or `"google/flan-t5-small"`).

3. **Run the Test Script**

   ```bash
   source ~/venv-llm/bin/activate   # ensure virtualenv is active
   python3 ~/lab_data/test_llm.py
   ```

   * You’ll see printed steps and finally something like:

     ```
     Loading tokenizer for huggyllama/llama-7b-hf-cpu...
     Loading model huggyllama/llama-7b-hf-cpu (CPU-only inference)...
     Tokenizing prompt...
     Generating up to 30 new tokens...

     ===== GENERATED OUTPUT =====

     In a cozy room, two friends talk about security. The first friend says, “We should always encrypt our data.” The second friend replies, “Absolutely—good encryption practices are vital to prevent breaches.”

     ============================
     ```
   * **Performance Note**: On a laptop with 12 GB RAM allocated to VM, loading the 7 B model may take 2–4 minutes and generation can take 30–60 seconds. Be patient or switch to a smaller model if needed.

4. **If You Encounter an OOM (Out of Memory) Error**

   * **Reduce Model Size**:

     ```python
     model_name = "huggyllama/llama-3b-hf-cpu"
     ```
   * **Increase VM Memory**:
     ‒ In VMware: Power off the VM → Right-click VM → **Settings → Memory** → increase to 16 GB or 20 GB (if host can spare).
   * **Use 4-bit Quantization on CPU** (Advanced):
     ‒ Some LLaMA variants support CPU-only quantized weights. E.g., `"TheBloke/Llama-7B-4bit-CPU"`—you’d do:

     ```bash
     pip install bitsandbytes  # required for 4-bit
     ```

     Then in code:

     ```python
     from transformers import AutoModelForCausalLM, AutoTokenizer

     model_name = "TheBloke/Llama-7B-4bit-CPU"
     tokenizer = AutoTokenizer.from_pretrained(model_name)
     model = AutoModelForCausalLM.from_pretrained(
         model_name,
         load_in_4bit=True,           # load quantized 4-bit weights
         device_map="cpu"
     )
     ```

### 5.4. Organize Local Model Caching (Optional)

By default, Hugging Face caches models in `~/.cache/huggingface/transformers` which can clutter your home directory. You can redirect it to a custom folder.

1. **Choose a Custom Cache Location**

   * For example: `~/models/huggingface/transformers`.
   * Also set `HF_HOME` so logs, token files, and config go under `~/models/huggingface`.

2. **Set Environment Variables**
   Add the following lines to your `~/.bashrc` (or `~/.profile`):

   ```bash
   export HF_HOME=~/models/huggingface
   export TRANSFORMERS_CACHE=$HF_HOME/transformers
   ```

   * **Why?**
     ‒ Keeps large model files outside of `~/.cache`, making cleanup and backups easier.
   * Save `~/.bashrc` and then run:

     ```bash
     source ~/.bashrc
     ```

3. **Verify Custom Cache**

   * Remove old cache (if you want to start fresh):

     ```bash
     rm -rf ~/.cache/huggingface
     ```
   * Re-run `test_llm.py` (or another model load). You should see new files appear under `~/models/huggingface/transformers/*`.

### 5.5. Install and Configure VS Code for AI Development

Using VS Code makes writing Python scripts (LLM or Neo4j integration) easier with syntax highlighting, extensions, and debugging support.

1. **Install VS Code via Snap**

   ```bash
   sudo snap install --classic code
   ```

   * **Why Snap?**
     ‒ Ensures you get the latest release directly from Microsoft.
     ‒ Automatically updates when a new VS Code version is available.

2. **Launch VS Code**

   ```bash
   code
   ```

   * If you see the splash and the editor window, VS Code is installed correctly.

3. **Install AI-Assisted Coding Extensions**

   * In VS Code, click the **Extensions** icon (left sidebar) or press **Ctrl + Shift + X**.
   * Search for and install:

     * **GitHub Copilot** (requires a Copilot subscription and GitHub account).
     * **TabNine** (offers a free tier—good AI code completions).
     * **Compass AI** (if your organization provides a license).
   * **Why Extensions?**
     ‒ Suggest code snippets (e.g., `AutoModelForCausalLM.from_pretrained(...)`) as you type.
     ‒ Help you discover library functions without constantly Googling.
     ‒ Increase coding speed and reduce typos.

4. **Configure Python Linting & Formatting (Optional but Recommended)**

   * Install linters/formatters in your virtualenv:

     ```bash
     pip install flake8 black
     ```
   * In VS Code’s **Settings** (Ctrl + ,):

     * Search `python.linting.enabled` → check it.
     * Choose `flake8` as the linter.
     * Search `python.formatting.provider` → set to `black`.
     * Optionally enable “Format on Save” so VS Code auto-formats with Black each time you save.
   * **Why?**
     ‒ Keeps your Python code consistent and free of common style issues.
     ‒ Helps you learn idiomatic Python style if you’re not already familiar.

### 5.6. (Optional) Explore a Local Chat UI

If you want an interactive chat interface rather than running scripts every time, you can deploy a minimal web-based UI for your local LLM. This is optional and depends on interest.

1. **Choose a “Chat Server”**

   * **FastChat** by [lm-sys/FastChat](https://github.com/lm-sys/FastChat).
   * **Text Generation Inference (TGI)** by Hugging Face (more complex to set up).
   * **llama.cpp** + **cTranslate2** (for extremely lightweight CPU inference).

2. **Install FastChat (Example)**

   ```bash
   source ~/venv-llm/bin/activate
   pip install fastchat
   ```

   * **Why FastChat?**
     ‒ Provides a simple REST API (port 8000) to send chat completions.
     ‒ Supports CPU inference via cTranslate2, which can be faster than PyTorch for some models.

3. **Download a Model for FastChat**

   * If you have a GPU and want to test LLaMA-2:

     1. Accept Meta’s license on Hugging Face for **`meta-llama/Llama-2-7b-chat-hf`**.
     2. Run:

        ```bash
        fastchat-models download --model llama2-7b-chat
        ```
   * If you have no GPU, choose a CPU-friendly model like **`huggyllama/llama-7b-hf-cpu`**. FastChat can serve CPU models too.

4. **Start FastChat Server**

   ```bash
   fastchat serve --model llama2-7b-chat --device cpu
   ```

   * **Output**:
     ‒ REST server listening on `http://0.0.0.0:8000`.
     ‒ You can send JSON POST requests to `/chat/completions`.

5. **Test a Chat Request**
   In Ubuntu VM’s terminal (install `jq` for pretty JSON if desired):

   ```bash
   curl -X POST http://localhost:8000/chat/completions \
     -H "Content-Type: application/json" \
     -d '{
           "model": "llama2-7b-chat",
           "messages": [
             {"role": "system", "content": "You are a helpful assistant."},
             {"role": "user", "content": "Tell me about Neo4j and Docker."}
           ]
         }'
   ```

   * **Expected Response**: A JSON object with `"choices"[0]["message"]["content"]` containing the assistant’s reply.

6. **(Optional) Deploy a Web Frontend**

   * FastChat’s repository includes a simple React-based frontend.
   * Clone it, build it with `npm install && npm run build`, then point it at `http://localhost:8000`.
   * Result: A browser-based chat window to experiment with your local LLM.

---

## 6. Ingest Sample Data into Neo4j

With Neo4j running in Docker and the Python/LLM environment ready, let’s load a tiny dataset of “people” and “friendships” into Neo4j. This proves the ETL (Extract, Transform, Load) pipeline from CSV → graph.

### 6.1. Prepare Sample CSV Files

1. **Create a Data Folder**

   ```bash
   mkdir -p ~/lab_data
   cd ~/lab_data
   ```

   * `~/lab_data` holds CSVs and Python scripts for ingestion and querying.

2. **Create `people.csv`**

   ```bash
   nano people.csv
   ```

   Paste:

   ```csv
   name:ID,age:int,city
   Alice,30,Seattle
   Bob,25,Denver
   Carol,28,San Francisco
   ```

   * **Header Explanation**:
     ‒ `name:ID` → Tells Neo4j this column is a unique identifier for nodes.
     ‒ `age:int` → Casts `age` as an integer in the graph.
     ‒ `city` → Defaults to a string property.

   Save & exit: in nano, Ctrl + O, Enter, Ctrl + X.

3. **Create `friendships.csv`**

   ```bash
   nano friendships.csv
   ```

   Paste:

   ```csv
   :START_ID,:END_ID
   Alice,Bob
   Bob,Carol
   Alice,Carol
   ```

   * **Header Explanation**:
     ‒ `:START_ID` and `:END_ID` refer to `name:ID` from `people.csv`.
     ‒ Neo4j will match existing nodes by `name` when creating relationships.

   Save & exit.

4. **Verify Files**

   ```bash
   ls -l
   # Should show: people.csv  friendships.csv
   ```

### 6.2. Install the Neo4j Python Driver

1. **Activate Virtualenv**

   ```bash
   source ~/venv-llm/bin/activate
   ```

2. **Install `neo4j` Python Package**

   ```bash
   pip install neo4j
   ```

   * **Why?**
     ‒ Provides the `GraphDatabase` class to create sessions and run Cypher queries from Python.

3. **Verify Installation**

   ```bash
   python3 -c "import neo4j; print('Neo4j driver version:', neo4j.__version__)"
   ```

   * Should print the installed driver version (e.g., `Neo4j driver version: 4.x.x`).

### 6.3. Write and Run the Ingestion Script

1. **Create `etl_to_neo4j.py`**

   ```bash
   nano ~/lab_data/etl_to_neo4j.py
   ```

2. **Paste the Following Code**

   ```python
   from neo4j import GraphDatabase
   import csv

   # Step A: Connection parameters to Neo4j (running in Docker on localhost)
   uri = "bolt://localhost:7687"
   auth = ("neo4j", "StrongPassword123")  # Replace with your updated Neo4j password

   driver = GraphDatabase.driver(uri, auth=auth)

   def ingest():
       with driver.session() as session:
           # (Optional) Clear existing data—useful for repeated tests
           session.run("MATCH (n) DETACH DELETE n")

           # Ingest Person nodes
           with open("people.csv", newline="") as f:
               reader = csv.DictReader(f)
               for row in reader:
                   session.run(
                       """
                       CREATE (p:Person {
                         name: $n,
                         age: toInteger($a),
                         city: $c
                       })
                       """,
                       n=row["name:ID"],
                       a=row["age:int"],
                       c=row["city"]
                   )

           # Ingest FRIEND_OF relationships
           with open("friendships.csv", newline="") as f:
               reader = csv.DictReader(f)
               for row in reader:
                   session.run(
                       """
                       MATCH (a:Person {name: $s})
                       MATCH (b:Person {name: $e})
                       CREATE (a)-[:FRIEND_OF]->(b)
                       """,
                       s=row[":START_ID"],
                       e=row[":END_ID"]
                   )

   if __name__ == "__main__":
       ingest()
       print("Graph ingestion complete.")
       driver.close()
   ```

   **Clarifications**:

   * **`session.run("MATCH (n) DETACH DELETE n")`**
     ‒ Deletes all nodes and relationships in the database. Useful for testing multiple times. Omit in production if you want to keep existing data.
   * **`CREATE (p:Person { ... })`**
     ‒ Creates a new node with label `Person` and properties `name`, `age`, `city`.
     ‒ `toInteger($a)` casts the `age` string from CSV to an integer.
   * **Relationship Ingestion**:
     ‒ We `MATCH` existing `Person` nodes by `name` (the `:START_ID` and `:END_ID` fields correspond to `name:ID` in `people.csv`).
     ‒ Then `CREATE (a)-[:FRIEND_OF]->(b)`.

3. **Save & Exit** (Ctrl + O, Enter, Ctrl + X).

4. **Run the Script**

   ```bash
   cd ~/lab_data
   python3 etl_to_neo4j.py
   ```

   * Expect “Graph ingestion complete.”
   * If you see errors like “Authentication failed,” confirm that the password matches the one you set in Neo4j Browser.
   * If “Connection refused,” check that the `neo4j-lab` container is running (`docker ps`) and listening on port 7687.

5. **Verify Data in Neo4j Browser**

   * In Ubuntu VM’s browser, go to `http://localhost:7474`.
   * Log in (Neo4j asks for a new password if you haven’t changed it already).
   * In the command prompt at the top, run:

     ```cypher
     MATCH (p:Person)-[:FRIEND_OF]->(f)
     RETURN p.name AS Person, collect(f.name) AS Friends;
     ```
   * You should see a table like:

     ```
     ┌────────┬───────────────────┐
     │ Person │ Friends           │
     ├────────┼───────────────────┤
     │ "Alice"│ ["Bob","Carol"]   │
     │ "Bob"  │ ["Carol"]         │
     └────────┴───────────────────┘
     ```

---

## 7. Integrate the LLM with Neo4j (Query + Summarize)

Now that Neo4j holds our sample graph, write a Python script that:

1. Queries Neo4j for a node’s friends.
2. Feeds that list to the LLM prompt.
3. Prints a one-sentence summary.

### 7.1. Create `query_and_summarize.py`

1. **Open a New File**

   ```bash
   nano ~/lab_data/query_and_summarize.py
   ```

2. **Paste the Following Code**

   ```python
   from neo4j import GraphDatabase
   from transformers import AutoModelForCausalLM, AutoTokenizer

   # (A) Neo4j connection parameters
   uri = "bolt://localhost:7687"
   auth = ("neo4j", "StrongPassword123")  # Replace with your password
   driver = GraphDatabase.driver(uri, auth=auth)

   def get_friends(tx, name):
       """
       Given a person’s name, fetch all friends’ names from Neo4j.
       """
       query = """
       MATCH (p:Person {name: $name})-[:FRIEND_OF]->(f)
       RETURN f.name AS friend
       """
       result = tx.run(query, name=name)
       return [record["friend"] for record in result]

   if __name__ == "__main__":
       # Step 1: Query Neo4j for Alice’s friends
       with driver.session() as session:
           friends = session.read_transaction(get_friends, "Alice")

       if not friends:
           print("No friends found for Alice.")
           driver.close()
           exit()

       # Step 2: Load a CPU-based LLM
       model_name = "huggyllama/llama-7b-hf-cpu"
       print(f"Loading model '{model_name}' (this may take a minute)...")
       tokenizer = AutoTokenizer.from_pretrained(model_name)
       model = AutoModelForCausalLM.from_pretrained(model_name)

       # Step 3: Build a prompt and run inference
       friend_list_str = ", ".join(friends)
       prompt = f"Alice is friends with {friend_list_str}. Summarize this relationship in one sentence."
       inputs = tokenizer(prompt, return_tensors="pt")

       print("Generating summary...")
       outputs = model.generate(inputs.input_ids, max_new_tokens=30)
       summary = tokenizer.decode(outputs[0], skip_special_tokens=True)

       # Step 4: Print results
       print("=== Neo4j Query Result ===")
       print(f"Alice’s friends: {friends}")
       print("=== LLM-Generated Summary ===")
       print(summary)

       driver.close()
   ```

   **Clarifications**:

   * **`get_friends`**: Uses a Cypher `MATCH` to retrieve names of nodes connected by `FRIEND_OF`.
   * **LLM Loading**: We re-use `"huggyllama/llama-7b-hf-cpu"`. If you run out of memory, switch to a smaller CPU model.
   * **Prompt Construction**: Embeds `"Alice is friends with Bob, Carol"`—this is fed verbatim to the model. You can modify the template as needed.
   * **`model.generate(...)`**: Produces new tokens. We limit to 30 tokens for brevity.

3. **Save & Exit** (Ctrl + O, Enter, Ctrl + X).

### 7.2. Run and Validate the Script

1. **Activate Virtualenv**

   ```bash
   source ~/venv-llm/bin/activate
   ```

2. **Run the Script**

   ```bash
   python3 ~/lab_data/query_and_summarize.py
   ```

   * Monitor console output—expect:

     ```
     Loading model 'huggyllama/llama-7b-hf-cpu' (this may take a minute)...
     Generating summary...
     === Neo4j Query Result ===
     Alice’s friends: ['Bob', 'Carol']
     === LLM-Generated Summary ===
     Alice is friends with Bob and Carol, forming a close-knit trio of companions.
     ```
   * **If It Errors**:
     ‒ **“Out of memory”**: Switch model to `llama-3b-hf-cpu` or `flan-t5-small`.
     ‒ **“Connection refused”**: Verify Neo4j container is up (`docker ps`).
     ‒ **Authentication errors**: Confirm the password in `auth = ("neo4j", "…")` matches Neo4j Browser’s.

3. **Experiment with Other Prompts**

   * Modify the prompt template to ask for a more detailed description:

     ```python
     prompt = f"Alice has the following friends: {friend_list_str}. Describe their relationships and what they enjoy doing together."
     ```
   * Rerun the script and observe how the generated text changes.

---

## 8. (Optional) Enable GPU Passthrough for Faster Inference

If your laptop has a discrete NVIDIA GPU and VMware Workstation Pro supports partial passthrough (rare on laptops), follow these steps. Otherwise, skip to Section 9.

### 8.1. Verify GPU Visibility in the Ubuntu VM

1. **Install NVIDIA Utilities** (if not already present)

   ```bash
   sudo apt install -y nvidia-utils-525
   ```

   * This provides `nvidia-smi` and basic utilities.
2. **Check with `nvidia-smi`**

   ```bash
   nvidia-smi
   ```

   * **Expected**: A table showing your GPU name, driver version, and memory usage.
   * **If You See “NVIDIA-SMI has failed”**:
     ‒ Your VM does not have GPU access.
     ‒ Revert to CPU-only inference (Section 5).

### 8.2. Install NVIDIA Drivers and CUDA Toolkit

1. **Install NVIDIA Driver**

   ```bash
   sudo apt install -y nvidia-driver-525
   sudo reboot
   ```

   * After reboot, run `nvidia-smi` again to confirm proper driver installation.
2. **Install CUDA Toolkit**

   ```bash
   sudo apt install -y nvidia-cuda-toolkit
   ```

   * Provides `nvcc` and CUDA libraries. May install an older CUDA version (e.g., 11.1).
   * For newer CUDA versions, follow NVIDIA’s .deb repository instructions:

     ```bash
     wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-ubuntu2204.pin
     sudo mv cuda-ubuntu2204.pin /etc/apt/preferences.d/cuda-repository-pin-600
     wget https://developer.download.nvidia.com/compute/cuda/12.2.0/local_installers/cuda-repo-ubuntu2204-12-2-local_12.2.0-535.86.10-1_amd64.deb
     sudo dpkg -i cuda-repo-ubuntu2204-12-2-local_12.2.0-535.86.10-1_amd64.deb
     sudo apt-key add /var/cuda-repo-ubuntu2204-12-2-local/7fa2af80.pub
     sudo apt update
     sudo apt install -y cuda
     ```
   * **Verify**:

     ```bash
     nvcc --version
     ```

     Should show the installed CUDA version.

### 8.3. Reinstall PyTorch with CUDA Support

1. **Activate Virtualenv**

   ```bash
   source ~/venv-llm/bin/activate
   ```
2. **Uninstall CPU-Only PyTorch**

   ```bash
   pip uninstall -y torch torchvision
   ```
3. **Install CUDA-Enabled PyTorch**

   * Find the correct “wheel URL” for your CUDA version at [pytorch.org](https://pytorch.org/get-started/locally/). For example, for CUDA 11.8:

     ```bash
     pip install --upgrade pip
     pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118
     pip install transformers accelerate bitsandbytes
     ```
   * **Why `accelerate` & `bitsandbytes`?**
     ‒ `accelerate` helps manage device placement (CPU vs. GPU) automatically.
     ‒ `bitsandbytes` enables 4-bit or 8-bit quantization, reducing GPU memory usage—critical for large models.
4. **Confirm GPU-Enabled PyTorch**

   ```bash
   python3 - <<EOF
   import torch
   print("CUDA available:", torch.cuda.is_available())
   print("CUDA device count:", torch.cuda.device_count())
   print("CUDA device name:", torch.cuda.get_device_name(0) if torch.cuda.is_available() else "None")
   EOF
   ```

   * Should print `CUDA available: True`, device count ≥ 1, and GPU name.

### 8.4. Switch to a GPU-Quantized LLaMA-2 Model

1. **Choose a 4-bit Quantized Model**

   * Example: `"decapoda-research/llama-2-7b-hf-4bit"` on Hugging Face.
   * This model uses 4-bit weights (via `bitsandbytes`) to fit in \~6–7 GB GPU memory (instead of \~14 GB for full precision).

2. **Modify `query_and_summarize.py` to Use GPU**
   Open the file:

   ```bash
   nano ~/lab_data/query_and_summarize.py
   ```

   Replace the model-loading section with:

   ```python
   from transformers import AutoModelForCausalLM, AutoTokenizer
   import torch

   # (B) Load a GPU-optimized, 4-bit quantized LLaMA-2 7B
   model_name = "decapoda-research/llama-2-7b-hf-4bit"
   print(f"Loading GPU-optimized model '{model_name}' (4-bit quantized)...")
   tokenizer = AutoTokenizer.from_pretrained(model_name)

   model = AutoModelForCausalLM.from_pretrained(
       model_name,
       load_in_4bit=True,           # Use 4-bit quantization
       device_map="auto",           # Automatically place layers on GPU
       torch_dtype=torch.float16    # 16-bit precision for faster compute
   )

   # Step 3 (generate) changes:
   inputs = tokenizer(prompt, return_tensors="pt").input_ids.cuda()
   outputs = model.generate(inputs, max_new_tokens=30)
   summary = tokenizer.decode(outputs[0].cpu(), skip_special_tokens=True)
   ```

   **Clarifications**:

   * **`load_in_4bit=True`** uses `bitsandbytes` to load the model in a 4-bit representation—significantly reducing VRAM usage.
   * **`device_map="auto"`** tells Hugging Face’s Accelerate to allocate model layers across available GPUs and CPU if needed.
   * We explicitly move `inputs.input_ids` to CUDA (`.cuda()`) and bring outputs back to CPU (`outputs[0].cpu()`) for decoding.

3. **Save & Exit**.

4. **Run the GPU-Enabled Script**

   ```bash
   source ~/venv-llm/bin/activate
   python3 ~/lab_data/query_and_summarize.py
   ```

   * You should see much faster generation times. On a mid-range GPU (e.g., NVIDIA RTX 3060, 8 GB), inference might take \~5–10 seconds instead of 30–60 seconds on CPU.

---

## 9. Security, Maintenance, and Cleanup

As your local lab grows, it’s important to secure services, take snapshots, monitor resources, and back up data to avoid accidents.

### 9.1. Secure Neo4j (Docker Container)

1. **Change the Default Password**

   * During first login at `http://localhost:7474`, Neo4j prompts you to change from `StrongPassword123` to something stronger (e.g., `MyLabNeo4jP@ssw0rd`).
   * To change it again from a terminal, run:

     ```bash
     docker exec -it neo4j-lab cypher-shell -u neo4j -p <oldpassword> \
       "ALTER CURRENT USER SET PASSWORD FROM '<oldpassword>' TO '<newpassword>'"
     ```
   * **Why?**
     ‒ Default passwords are well-known; always change to avoid unauthorized access.

2. **Lock Down Network Access (If You Choose Bridged Mode)**

   * If you switch the VM’s network adapter to **Bridged**, other devices on your LAN can reach `http://<VM_IP>:7474`.
   * Limit Neo4j’s accessibility to “localhost” (only inside the VM) by enabling Ubuntu’s firewall (`ufw`):

     ```bash
     sudo apt install -y ufw
     sudo ufw default deny incoming
     sudo ufw default allow outgoing
     # Allow SSH from specific IP if you use SSH:
     # sudo ufw allow from 192.168.1.50 to any port 22
     # Allow Neo4j access only on localhost:
     sudo ufw allow from 127.0.0.1 to any port 7474
     sudo ufw allow from 127.0.0.1 to any port 7687
     sudo ufw enable
     ```
   * **Why?**
     ‒ Ensures that even if the VM is on the LAN, external devices cannot connect to Neo4j unless they’re SSH’ing into the VM and using port forwarding.

3. **Docker Container Updates**

   * Periodically pull the latest Neo4j image and recreate the container:

     ```bash
     docker compose pull
     docker compose down
     docker compose up -d
     ```
   * **Caveat**:
     ‒ If a new Neo4j major version changes features, your data may or may not be compatible. Always back up first.

4. **Backup Neo4j Data**

   * Your data folder is `~/ai_graph_lab/neo4j_data`. To back it up:

     ```bash
     cd ~/ai_graph_lab
     tar czf neo4j_data_backup_$(date +%F).tgz neo4j_data
     ```
   * To restore:

     ```bash
     docker compose down
     rm -rf neo4j_data
     tar xzf neo4j_data_backup_<date>.tgz
     docker compose up -d
     ```

### 9.2. Snapshot Your VM

1. **In VMware Workstation Pro**

   * Power off the VM.
   * Right-click `Ubuntu_AI_Lab` → **Snapshot → Take Snapshot**.
   * **Name**: “Baseline: Ubuntu + Docker + Neo4j + LLM (CPU)”
   * **Description**: “All components installed, Neo4j running, CPU LLM tested.”
   * Click **OK**.

2. **Why Snapshots?**

   * If you break your Python environment or Docker configuration, you can revert to this snapshot instantly.
   * You can take another snapshot after enabling GPU and GPU-enabled LLM and label it “CUDA LLM.”

3. **Managing Snapshot Size**

   * Snapshots store differential data. If you allocate 12 GB RAM, the VM’s memory snapshot can be large. If you only snapshot the **power-off** state (disk state, not memory), it’s smaller.
   * Delete old snapshots as needed to free host disk space.

### 9.3. Monitor Resources

1. **Inside the Ubuntu VM**

   * **`htop`**: Install if not present:

     ```bash
     sudo apt install -y htop
     htop
     ```

     ‒ Shows real-time CPU (per core), memory, and process usage.
   * **`docker stats`**:

     ```bash
     docker stats neo4j-lab
     ```

     ‒ Displays CPU %, memory usage, network I/O specific to the Neo4j container.

2. **If Your LLM Script Consumes Too Much RAM/CPU**

   * Reduce model size (use 3 B instead of 7 B).
   * Limit concurrency: only run one generation at a time.
   * Reboot VM to clear memory.

3. **VMware Tools (Optional but Helpful)**

   * Install VMware Tools to improve VM performance and allow for better resource integration (copy/paste, improved graphics). In Ubuntu:

     ```bash
     sudo apt install -y open-vm-tools open-vm-tools-desktop
     sudo reboot
     ```
   * After reboot, you can monitor CPU & memory usage of the VM from VMware’s UI.

### 9.4. Clean Up Unused Docker Images & Containers

1. **List All Docker Containers (Including Stopped)**

   ```bash
   docker ps -a
   ```
2. **Remove Unused Containers**

   ```bash
   # Remove a specific container (if not needed)
   docker rm <container_id>
   # Or remove _all_ stopped containers:
   docker container prune
   ```
3. **List All Docker Images**

   ```bash
   docker images
   ```
4. **Remove Unused Images**

   ```bash
   docker image prune      # Removes dangling (unreferenced) images
   docker image prune -a   # Removes all images not used by existing containers
   ```

   * **Why?**
     ‒ Frees tens of gigabytes if you pulled multiple versions of Neo4j or test LLM containers.

### 9.5. Archive Your Python Environment (Optional)

1. **Export Installed Packages**

   ```bash
   source ~/venv-llm/bin/activate
   pip freeze > ~/lab_data/requirements.txt
   ```

   * **Why?**
     ‒ If you or a colleague want to replicate the exact Python environment on another machine, simply run:

     ```bash
     python3 -m venv venv-llm
     source venv-llm/bin/activate
     pip install -r requirements.txt
     ```

2. **Archive Your Entire VM (Optional)**

   * In VMware, you can export the VM as an **OVF**:
     ‒ **File → Export to OVF** → choose a destination.
     ‒ Result is a set of `.ovf` + `.vmdk` files you can import on another VMware host.

---

## 10. Detailed Summary & Next Steps

Below is a concise, high-level checklist referencing each detailed section above. Use this to review or guide your setup from start to finish.

1. **Host Machine Preparation (Section 1)**

   * Confirm ≥ 16 GB RAM, CPU virtualization (VT-x/AMD-V), and ≥ 200 GB free disk.
   * Install/update VMware Workstation Pro (or Fusion Pro).
   * Create a host folder, e.g., `~/VMs/AI_Lab`.
   * Use NAT networking for the VM.

2. **Create Ubuntu VM (Section 2)**

   1. **New VM → Typical → Ubuntu 22.04 ISO**
   2. **Hardware Customization**: 4 vCPUs, 12 GB RAM, 100 GB disk (thin), NAT, 3D Graphics enabled.
   3. **Install Ubuntu**:

      * Name user `labuser`, set a strong password.
      * Install third-party software, updates after install.
   4. **Post-Install**:

      * `sudo apt update && sudo apt upgrade -y`
      * `sudo apt install build-essential git curl wget python3 python3-venv python3-pip`
      * `sudo reboot`

3. **Install Docker & Docker Compose (Section 3)**

   * Run: `curl -fsSL https://get.docker.com | sudo sh`
   * Add user to `docker` group: `sudo usermod -aG docker $USER` → log out/in.
   * Install Compose plugin: `sudo apt install docker-compose-plugin`
   * Verify with `docker version` and `docker compose version`.

4. **Deploy Neo4j via Docker Compose (Section 4)**

   1. `mkdir ~/ai_graph_lab && cd ~/ai_graph_lab`
   2. Create `docker-compose.yml` with Neo4j 4.5, ports 7474/7687, `neo4j_data` volume.
   3. `docker compose up -d` → confirm with `docker ps`.
   4. In Ubuntu browser, go to `http://localhost:7474` → login & change password.
   5. Verify data directory: `ls ~/ai_graph_lab/neo4j_data`.

5. **Set Up LLM/Inference Environment (Section 5)**

   1. `python3 -m venv ~/venv-llm` → `source ~/venv-llm/bin/activate`
   2. `pip install --upgrade pip`
   3. `pip install torch torchvision transformers sentencepiece`
   4. Create and run `~/lab_data/test_llm.py` with `"huggyllama/llama-7b-hf-cpu"`.
   5. (Optional) Set `HF_HOME` & `TRANSFORMERS_CACHE` in `~/.bashrc` for custom cache.
   6. Install VS Code via `snap install --classic code` → install Copilot/TabNine/Compass AI.
   7. (Optional) Deploy FastChat for a local chat UI: `pip install fastchat` → `fastchat serve --model llama2-7b-chat --device cpu`.

6. **Ingest Sample Data into Neo4j (Section 6)**

   1. `mkdir ~/lab_data && cd ~/lab_data`
   2. Create `people.csv` and `friendships.csv` with appropriate headers.
   3. `source ~/venv-llm/bin/activate` → `pip install neo4j`
   4. Write `etl_to_neo4j.py` → `python3 etl_to_neo4j.py`
   5. Verify via Neo4j Browser: `MATCH (p:Person)-[:FRIEND_OF]->(f) RETURN p.name, collect(f.name)`.

7. **Integrate the LLM with Neo4j (Section 7)**

   1. Write `query_and_summarize.py` (Neo4j query → LLM prompt → print summary).
   2. Run `python3 ~/lab_data/query_and_summarize.py` under virtualenv.
   3. Modify prompts or swap models to experiment.

8. **Optional GPU Passthrough (Section 8)**

   1. Check GPU presence: `nvidia-smi`.
   2. Install driver: `sudo apt install nvidia-driver-525` → `sudo reboot`.
   3. Install CUDA Toolkit: `sudo apt install nvidia-cuda-toolkit`.
   4. Reinstall PyTorch with CUDA:

      ```bash
      source ~/venv-llm/bin/activate
      pip uninstall -y torch torchvision
      pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118
      pip install transformers accelerate bitsandbytes
      ```
   5. Modify `query_and_summarize.py` to load `"decapoda-research/llama-2-7b-hf-4bit"` with `load_in_4bit=True`, `device_map="auto"`.
   6. Run GPU-enabled script and confirm faster inference.

9. **Security, Maintenance, Cleanup (Section 9)**

   1. Change Neo4j default password in the browser or via `docker exec neo4j-lab`.
   2. If using Bridged networking, enable UFW to block external access to ports 7474/7687.
   3. Take VMware snapshot: “Baseline: Ubuntu + Docker + Neo4j + LLM (CPU)”.
   4. Monitor resources: `htop`, `docker stats`.
   5. Prune unused Docker containers/images: `docker container prune`, `docker image prune -a`.
   6. Backup Neo4j data: `tar czf neo4j_data_backup_$(date +%F).tgz neo4j_data`.
   7. Export Python environment: `pip freeze > ~/lab_data/requirements.txt`.

---

### Final Tips & Reminders

* **Always Activate the Virtualenv** before running any Python scripts. Look for `(venv-llm)` in your prompt.
* **Model Selection**: Start with CPU models, then move to GPU-enabled if your hardware supports it. For CPU-only use smaller models or 4-bit quantized variants.
* **Neo4j Persistence**: The folder `~/ai_graph_lab/neo4j_data` holds the graph. If you remove or rename it, Neo4j will start with an empty database next time.
* **Snapshots Are Your Friend**: Before making big changes (e.g., upgrading Docker, reinstalling CUDA), take a snapshot so you can revert if things break.
* **Resource Contention**: Running Neo4j, Python inference, and VS Code all at once can saturate CPU/RAM. Close heavy applications if you see performance issues.
* **Port Mapping vs. Host Access**: Because the VM uses NAT, services published on `localhost` inside the VM are reachable from the host as `localhost:<port>`. If you switch to Bridged, you must adjust firewall rules.
* **Clean Up Frequently**: Docker images and containers, as well as Hugging Face model caches, can consume tens of gigabytes. Use `docker image prune -a` and occasionally delete old model caches.
* **Practice & Experiment**: Modify CSVs, add new `Person` nodes, create other relationship types. Swap LLM models. Build a simple web front end (Flask/Streamlit) to integrate Neo4j queries + LLM summarization in a UI.

By following these detailed steps (with rationale), you’ll not only have a working local lab for **Neo4j + LLM inference** but also understand the “why” behind each decision. This knowledge empowers you to adapt and extend the setup—swap in different databases (e.g., Postgres + PGGraph), test other transformer-based models, or containerize your own microservices for a fuller AI/knowledge-graph pipeline. Good luck, and enjoy learning as you build!
