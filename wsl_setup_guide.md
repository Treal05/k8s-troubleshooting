h1. Setting Up Your WSL Environment for Kubernetes Troubleshooting

This guide will walk you through setting up your Windows Subsystem for Linux (WSL) environment with Ubuntu LTS, updating it, and installing the necessary tools for Kubernetes troubleshooting.

h2. Step 1: Enable WSL Feature

1. Open PowerShell as Administrator.
2. Run the following command to enable the Windows Subsystem for Linux feature. This command will not install a Linux distribution.
{code:bash}
wsl --install --no-distribution
{code}
3. Restart your computer when prompted.

h2. Step 2: Install Ubuntu from the Microsoft Store

1. Open the Microsoft Store application on your Windows machine.
2. Search for "Ubuntu 24.04 LTS" (or the latest LTS version available).
3. Click "Get" or "Install" to download and install the Ubuntu application.
4. Once installed, you can launch Ubuntu from the Start Menu.

h2. Step 3: Initial Ubuntu Setup

1. On the first launch, Ubuntu will complete its installation and then prompt you to create a username and password. This will be your user for the WSL environment.

h2. Step 4: Update and Upgrade Ubuntu Packages

1. Open your Ubuntu terminal.
2. Run the following commands to update the package lists and upgrade all installed packages to their latest versions:
{code:bash}
sudo apt update && sudo apt upgrade -y
{code}

h2. Step 5: Install and Supercharge Zsh (Optional but Recommended)

This step enhances your terminal experience with Zsh, a powerful shell, and various plugins and themes.

1.  **Install Zsh:**
    {code:bash}
    sudo apt install zsh -y
    {code}
    _Explanation:_ Zsh (Z Shell) is an extended Bourne shell with many improvements, including better tab completion, theme support, and plugin architecture.

2.  **Set Zsh as Default Shell:**
    {code:bash}
    chsh -s $(which zsh)
    {code}
    _Explanation:_ This command changes your default login shell to Zsh. You will need to close and reopen your terminal for this change to take effect.

3.  **Install Oh My Zsh:**
    {code:bash}
    sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended
    {code}
    _Explanation:_ Oh My Zsh is a delightful, open-source framework for managing your Zsh configuration. It comes with a vast collection of plugins and themes.

4.  **Install Powerlevel10k Theme:**
    {code:bash}
    git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
    sed -i 's/ZSH_THEME="robbyrussell"/ZSH_THEME="powerlevel10k\/powerlevel10k"/' ~/.zshrc
    {code}
    _Explanation:_ Powerlevel10k is a highly customizable and fast Zsh theme that provides a modern and informative prompt.

5.  **Install Zsh Plugins (Autosuggestions, Syntax Highlighting, fzf):**
    {code:bash}
    git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
    git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
    sudo apt install fzf -y
    {code}
    _Explanation:_
    *   `zsh-autosuggestions`: Suggests commands as you type based on your history.
    *   `zsh-syntax-highlighting`: Highlights commands as you type, making it easier to spot errors.
    *   `fzf`: A general-purpose command-line fuzzy finder, useful for quickly searching history, files, and processes.

6.  **Enable Plugins in .zshrc:**
    {code:bash}
    sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions zsh-syntax-highlighting fzf)/' ~/.zshrc
    {code}
    _Explanation:_ This command modifies your `.zshrc` file to activate the newly installed plugins.

h2. Step 6: Install Kubernetes Tools

This step installs essential command-line tools for interacting with Kubernetes clusters.

1.  **Install kubectl:**
    {code:bash}
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    rm kubectl
    {code}
    _Explanation:_ `kubectl` is the official command-line tool for running commands against Kubernetes clusters. You use it to deploy applications, inspect and manage cluster resources, and view logs.

2.  **Configure Kubeconfig Files:**

    After installing `kubectl`, you'll need to configure it to connect to your Kubernetes clusters. In our environment, `kubeconfig` files are typically downloaded from Rancher and saved in your `~/.kube/` directory with names like `cluster-1.yaml`, `cluster-2.yaml`, etc.

    *   **Create the .kube directory (if it doesn't exist):**
        {code:bash}
        mkdir -p ~/.kube
        {code}
        _Explanation:_ This command creates the `.kube` directory in your home folder, where `kubectl` expects to find configuration files. The `-p` flag ensures that the directory is created only if it doesn't already exist, and it won't throw an error if it does.

    *   **Place your kubeconfig files:**
        Download your `kubeconfig` files from Rancher and save them into the `~/.kube/` directory (e.g., `~/.kube/cluster-1.yaml`, `~/.kube/cluster-2.yaml`).

    *   **Add all kubeconfig files to KUBECONFIG environment variable:**
        To allow `kubectl` to use multiple `kubeconfig` files simultaneously, you can set the `KUBECONFIG` environment variable to a colon-separated list of paths to your `kubeconfig` files.

        For the current session:
        {code:bash}
        export KUBECONFIG=~/.kube/config:$(find ~/.kube/ -maxdepth 1 -name 'cluster-*.yaml' -print0 | xargs -0 echo | tr ' ' ':')
        {code}
        _Explanation:_ This command sets the `KUBECONFIG` environment variable. It includes the default `~/.kube/config` file and then dynamically finds all files matching `cluster-*.yaml` within the `~/.kube/` directory, joining them with colons. This allows `kubectl` to access all specified cluster configurations.

        To make this persistent for Zsh (add to `~/.zshrc`):
        {code:bash}
        echo "export KUBECONFIG=~/.kube/config:$(find ~/.kube/ -maxdepth 1 -name '*.yaml' -print0 | xargs -0 echo | tr ' ' ':')" >> ~/.zshrc
        {code}
        _Explanation:_ This command appends the `export KUBECONFIG` line to your `~/.zshrc` file. This ensures that every time you open a new Zsh terminal, the `KUBECONFIG` environment variable is automatically set with all your cluster configurations, making them immediately available to `kubectl`.

3.  **Install Helm:**
    {code:bash}
    curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
    sudo apt-get install apt-transport-https --yes
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    sudo apt-get update
    sudo apt-get install helm -y
    {code}
    _Explanation:_ Helm is the package manager for Kubernetes. It helps you define, install, and upgrade even the most complex Kubernetes applications.

3.  **Install Kustomize:**
    {code:bash}
    curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
    sudo mv kustomize /usr/local/bin/
    {code}
    _Explanation:_ Kustomize is a tool for customizing Kubernetes configurations without templating. It allows you to manage multiple variations of your application configuration.

4.  **Install k9s(Additional Tools):**
    {code:bash}
    curl -sS https://webi.sh/k9s | sh
    source ~/.config/envman/PATH.env
    {code}
    _Explanation:_
    *   `k9s`: A terminal UI for Kubernetes clusters, providing a visual way to navigate, observe, and manage your applications.

h2. Step 7: Open WSL in Visual Studio Code

Visual Studio Code (VS Code) has excellent integration with WSL, allowing you to develop directly within your Linux environment.

1.  **Install "Remote - WSL" Extension:**
    *   Open VS Code on your Windows machine.
    *   Go to the Extensions view (Ctrl+Shift+X).
    *   Search for "Remote - WSL" and install it.

2.  **Open Your Project in WSL:**
    *   From your WSL Ubuntu terminal, navigate to your project directory (e.g., `cd ~/my-kubernetes-project`).
    *   Type `code .` and press Enter.
    *   _Explanation:_ This command will launch VS Code, connecting it to your WSL environment and opening the current directory. All terminal commands within VS Code will now run inside your WSL instance.


---

h1. Advanced Configuration and Usage

(Advanced configuration sections remain here)

---

h1. Combined Installation Script

This script combines all the necessary commands to set up your environment. You can save it as a `.sh` file (e.g., `setup.sh`) in your WSL environment and execute it with `bash setup.sh`.

{code:bash}
#!/bin/bash

# This script automates the setup of a Kubernetes troubleshooting environment on Ubuntu WSL.

# --- Update, Upgrade, and Install Prerequisites ---
echo "Updating and upgrading system packages..."
sudo apt update && sudo apt upgrade -y

echo "Installing git and curl..."
sudo apt install git curl -y

# --- Install and Configure Zsh ---
echo "Installing and configuring Zsh..."
sudo apt install zsh -y
# Set Zsh as default shell. This requires user to close and reopen the terminal later.
chsh -s $(which zsh)

# --- Install Oh My Zsh (non-interactive) ---
echo "Installing Oh My Zsh..."
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended

# --- Install Powerlevel10k Theme ---
echo "Installing Powerlevel10k theme..."
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
sed -i 's/ZSH_THEME="robbyrussell"/ZSH_THEME="powerlevel10k\/powerlevel10k"/' ~/.zshrc

# --- Install Zsh Plugins ---
echo "Installing Zsh plugins (autosuggestions and syntax-highlighting)..."
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# --- Install fzf ---
echo "Installing fzf..."
sudo apt install fzf -y

# --- Enable Plugins in .zshrc ---
echo "Enabling plugins in .zshrc..."
sed -i 's/plugins=(git)/plugins=(git zsh-autosuggestions zsh-syntax-highlighting fzf)/' ~/.zshrc

# --- Install Kubernetes Tools ---
echo "Installing kubectl..."
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# --- Configure Kubeconfig Files ---
echo "Creating ~/.kube directory and configuring KUBECONFIG..."
mkdir -p ~/.kube
echo "export KUBECONFIG=~/.kube/config:$(find ~/.kube/ -maxdepth 1 -name '*.yaml' -print0 | xargs -0 echo | tr ' ' ':')" >> ~/.zshrc

echo "Installing Helm..."
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm -y

echo "Installing Kustomize..."
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
sudo mv kustomize /usr/local/bin/

# --- Install Additional Tools ---
echo "Installing k9s..."
curl -sS https://webi.sh/k9s | sh
source ~/.config/envman/PATH.env

echo "\n--- SETUP COMPLETE ---"
echo "Please close and reopen your terminal to use your new Zsh shell."
echo "After restarting, you can run the verification script to check the installations."
{code}

h1. Verification Script

After running the setup script and restarting your terminal, you can run this script to verify that all tools were installed correctly. Save it as `verify.sh` and run it with `bash verify.sh`.

{code:bash}
#!/bin/bash

echo "--- Verifying installations ---"

# Function to check if a command exists and print its version
check_command() {
    if command -v $1 &> /dev/null; then
        echo -n "✅ $1: Installed | Version: "
        # Attempt to get version, handle commands that use -v, --version, or version
        if [[ "$1" == "kustomize" ]]; then
            $1 version | head -n 1
        elif [[ "$1" == "zsh" ]]; then
            $1 --version | head -n 1
        else
            $1 version 2>/dev/null || $1 --version 2>/dev/null || $1 -v 2>/dev/null
        fi
    else
        echo "❌ $1: NOT Installed"
    fi
}

check_command kubectl
check_command helm
check_command kustomize
check_command k9s
check_command zsh
check_command fzf

echo "\n--- Verification complete ---"
{code}
