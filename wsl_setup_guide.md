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

(Instructions for Zsh, Oh My Zsh, Powerlevel10k, and plugins remain here)

h2. Step 6: Install Kubernetes Tools

(Instructions for kubectl, helm, and kustomize remain here)

h2. Step 7: Open WSL in Visual Studio Code

(Instructions for opening WSL in VS Code remain here)

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
echo "Installing k9s and stern..."
curl -sS https://webi.sh/k9s | sh
source ~/.config/envman/PATH.env
curl -sS https://webi.sh/stern | sh
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
check_command stern
check_command zsh
check_command fzf

echo "\n--- Verification complete ---"
{code}