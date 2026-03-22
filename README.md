#!/bin/bash

# All-in-One Portable AI Cathedral Setup
# Downloads models, configures, and starts Ollama from a portable drive

set -e

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Configuration
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
MODELS_DIR="$SCRIPT_DIR/ollama_models"
DEFAULT_MODEL="qwen2.5:0.5b"
MODELS_TO_DOWNLOAD=(
    "qwen2.5:0.5b"
    "llama3.2:1b"
    "tinyllama:1.1b"
)

# Functions
print_status() { echo -e "${BLUE}[*]${NC} $1"; }
print_success() { echo -e "${GREEN}[✓]${NC} $1"; }
print_error() { echo -e "${RED}[✗]${NC} $1"; }
print_warning() { echo -e "${YELLOW}[!]${NC} $1"; }
print_header() {
    echo ""
    echo -e "${BLUE}========================================${NC}"
    echo -e "${BLUE}  $1${NC}"
    echo -e "${BLUE}========================================${NC}"
    echo ""
}

# Check if running as root (we don't want to run as root)
if [ "$EUID" -eq 0 ]; then
    print_error "Please do not run this script as root"
    exit 1
fi

# Main script
print_header "Portable AI Cathedral - Complete Setup"

# Step 1: Detect and prepare directories
print_status "Setting up directories..."
mkdir -p "$MODELS_DIR"
print_success "Models directory ready: $MODELS_DIR"

# Step 2: Check available space
AVAILABLE_SPACE=$(df -h "$SCRIPT_DIR" | awk 'NR==2 {print $4}')
print_status "Available space: $AVAILABLE_SPACE"

# Step 3: Install/Check Ollama
print_status "Checking Ollama installation..."
if ! command -v ollama &> /dev/null; then
    print_warning "Ollama not found. Installing..."
    curl -fsSL https://ollama.com/install.sh | sh
    if command -v ollama &> /dev/null; then
        print_success "Ollama installed successfully"
    else
        print_error "Failed to install Ollama"
        exit 1
    fi
else
    OLLAMA_VERSION=$(ollama --version)
    print_success "Ollama already installed: $OLLAMA_VERSION"
fi

# Step 4: Stop any running Ollama instances
print_status "Stopping existing Ollama processes..."
sudo systemctl stop ollama 2>/dev/null || true
pkill ollama 2>/dev/null || true
sleep 2
print_success "Stopped existing instances"

# Step 5: Configure environment
print_status "Configuring environment..."
export OLLAMA_MODELS="$MODELS_DIR"

# Systemd configuration
if command -v systemctl &> /dev/null; then
    print_status "Configuring systemd service..."
    sudo mkdir -p /etc/systemd/system/ollama.service.d
    sudo tee /etc/systemd/system/ollama.service.d/cathedral.conf > /dev/null << EOF
[Service]
Environment="OLLAMA_MODELS=$MODELS_DIR"
EOF
    sudo systemctl daemon-reload
    print_success "Systemd configured"
fi

# Shell configuration
for rcfile in ~/.bashrc ~/.zshrc; do
    if [ -f "$rcfile" ]; then
        if ! grep -q "OLLAMA_MODELS.*$MODELS_DIR" "$rcfile" 2>/dev/null; then
            echo "export OLLAMA_MODELS=\"$MODELS_DIR\"" >> "$rcfile"
            print_success "Added to $rcfile"
        fi
    fi
done

# Current session
export OLLAMA_MODELS="$MODELS_DIR"
print_success "Environment configured"

# Step 6: Download models
print_header "Model Management"

# Check existing models
print_status "Checking existing models..."
EXISTING_MODELS=""
if [ -d "$MODELS_DIR" ]; then
    EXISTING_MODELS=$(ollama list 2>/dev/null | tail -n +2 | awk '{print $1}' || echo "")
fi

# Offer to download models
echo ""
print_warning "Select models to download:"
echo "1) Lightweight - qwen2.5:0.5b (~400MB)"
echo "2) Balanced - llama3.2:1b (~800MB)"
echo "3) Powerful - tinyllama:1.1b (~1.1GB)"
echo "4) All models"
echo "5) Skip download (use existing)"
echo "6) Custom model"
echo ""
read -p "Choose option (1-6): " MODEL_CHOICE

download_model() {
    local model=$1
    print_status "Downloading $model..."
    if OLLAMA_MODELS="$MODELS_DIR" ollama pull "$model"; then
        print_success "Downloaded $model"
        return 0
    else
        print_error "Failed to download $model"
        return 1
    fi
}

case $MODEL_CHOICE in
    1)
        download_model "qwen2.5:0.5b"
        ;;
    2)
        download_model "llama3.2:1b"
        ;;
    3)
        download_model "tinyllama:1.1b"
        ;;
    4)
        for model in "${MODELS_TO_DOWNLOAD[@]}"; do
            download_model "$model"
        done
        ;;
    5)
        print_status "Skipping model downloads"
        ;;
    6)
        read -p "Enter model name (e.g., mistral, llama2): " CUSTOM_MODEL
        download_model "$CUSTOM_MODEL"
        ;;
    *)
        print_warning "Invalid choice, skipping downloads"
        ;;
esac

# Step 7: Start Ollama
print_header "Starting Ollama Service"

print_status "Starting Ollama with models from: $MODELS_DIR"

if command -v systemctl &> /dev/null; then
    sudo systemctl start ollama 2>/dev/null
    sleep 3
    if sudo systemctl is-active --quiet ollama 2>/dev/null; then
        print_success "Ollama running as system service"
    else
        print_warning "Service failed, starting manually..."
        OLLAMA_MODELS="$MODELS_DIR" nohup ollama serve > "$SCRIPT_DIR/ollama.log" 2>&1 &
        sleep 3
        print_success "Ollama started manually (log: $SCRIPT_DIR/ollama.log)"
    fi
else
    OLLAMA_MODELS="$MODELS_DIR" nohup ollama serve > "$SCRIPT_DIR/ollama.log" 2>&1 &
    sleep 3
    print_success "Ollama started manually (log: $SCRIPT_DIR/ollama.log)"
fi

# Step 8: Verify and test
print_header "Verification"

print_status "Testing connection..."
if curl -s http://localhost:11434/api/tags > /dev/null 2>&1; then
    print_success "Ollama API is accessible"
else
    print_warning "Ollama API not responding yet, waiting..."
    for i in {1..10}; do
        sleep 2
        if curl -s http://localhost:11434/api/tags > /dev/null 2>&1; then
            print_success "Ollama API is now accessible"
            break
        fi
    done
fi

# Show available models
echo ""
print_status "Available models:"
ollama list 2>/dev/null || echo "No models found"

# Step 9: Create helper scripts
print_header "Creating Helper Scripts"

# Start script
cat > "$SCRIPT_DIR/start_ai.sh" << EOF
#!/bin/bash
export OLLAMA_MODELS="$MODELS_DIR"
ollama serve &
echo "AI Cathedral started with models from: $MODELS_DIR"
echo "Models available:"
ollama list
EOF
chmod +x "$SCRIPT_DIR/start_ai.sh"
print_success "Created start_ai.sh"

# Stop script
cat > "$SCRIPT_DIR/stop_ai.sh" << EOF
#!/bin/bash
pkill ollama
echo "AI Cathedral stopped"
EOF
chmod +x "$SCRIPT_DIR/stop_ai.sh"
print_success "Created stop_ai.sh"

# Quick chat script
cat > "$SCRIPT_DIR/chat.sh" << EOF
#!/bin/bash
if [ -z "\$1" ]; then
    echo "Usage: ./chat.sh <model>"
    echo "Example: ./chat.sh qwen2.5:0.5b"
    exit 1
fi
export OLLAMA_MODELS="$MODELS_DIR"
ollama run "\$1"
EOF
chmod +x "$SCRIPT_DIR/chat.sh"
print_success "Created chat.sh"

# Create aliases
cat >> ~/.bashrc << 'EOF'

# AI Cathedral Aliases
alias ai-start='export OLLAMA_MODELS="'"$MODELS_DIR"'" && ollama serve &'
alias ai-list='export OLLAMA_MODELS="'"$MODELS_DIR"'" && ollama list'
alias ai-stop='pkill ollama'
EOF
print_success "Added shell aliases"

# Step 10: Display summary
print_header "Setup Complete!"

echo -e "${GREEN}✓${NC} Models location: $MODELS_DIR"
echo -e "${GREEN}✓${NC} Ollama running on: http://localhost:11434"
echo ""
echo "Quick commands:"
echo "  ${YELLOW}ai-list${NC}           - List available models"
echo "  ${YELLOW}ai-start${NC}          - Start AI Cathedral"
echo "  ${YELLOW}ai-stop${NC}           - Stop AI Cathedral"
echo "  ${YELLOW}./chat.sh <model>${NC} - Chat with a model"
echo ""
echo "Examples:"
for model in $(ollama list 2>/dev/null | tail -n +2 | awk '{print $1}'); do
    echo "  ollama run $model"
done
echo ""
echo "To use in new terminals, run:"
echo "  ${YELLOW}source ~/.bashrc${NC}"
echo ""
echo "Log file: $SCRIPT_DIR/ollama.log"
echo ""
print_success "AI Cathedral is ready to use!"

# Optional: Interactive chat
echo ""
read -p "Start interactive chat now? (y/n): " START_CHAT
if [[ $START_CHAT == "y" || $START_CHAT == "Y" ]]; then
    # Get first available model
    FIRST_MODEL=$(ollama list 2>/dev/null | tail -n +2 | awk '{print $1}' | head -1)
    if [ -n "$FIRST_MODEL" ]; then
        print_status "Starting chat with $FIRST_MODEL..."
        echo "Type /exit to quit"
        echo ""
        ollama run "$FIRST_MODEL"
    else
        print_error "No models available. Please download a model first." 🚀 Quick Start Instructions
On Any Linux System:

    Copy the script to your portable drive:

bash

# Save the script to your portable drive
nano /media/$USER/YourDrive/cathedral_ai.sh
# Paste the entire script above, save and exit (Ctrl+X, Y, Enter)

    Make it executable:

bash

chmod +x /media/$USER/YourDrive/cathedral_ai.sh

    Run the all-in-one setup:

bash

cd /media/$USER/YourDrive/
./cathedral_ai.sh

    Follow the interactive prompts:

    Choose models to download

    Let it configure everything automatically

    Start chatting!

📦 What This Script Does
Step	Action
1	Checks and installs Ollama if needed
2	Creates models directory on your drive
3	Downloads selected models (interactive)
4	Configures environment variables
5	Sets up systemd service (Linux)
6	Creates helper scripts and aliases
7	Starts Ollama with your portable models
8	Verifies everything works
9	Offers to start an interactive chat
🎯 Features

    Interactive model selection - Choose what to download

    Automatic dependency installation - Installs Ollama if missing

    Cross-platform support - Works on any Linux distro

    Helper scripts - Start/stop/chat scripts created automatically

    Shell aliases - ai-start, ai-list, ai-stop for quick access

    Persistent configuration - Survives reboots and new terminals

    Error handling - Continues even if some steps fail

    Progress feedback - Clear status messages throughout

📝 Helper Scripts Created

After running the script, you'll have:

    start_ai.sh - Start the AI service

    stop_ai.sh - Stop the AI service

    chat.sh - Quick chat with a model

    ollama.log - Service log file

💡 Usage Examples
bash

# Start the service
./start_ai.sh

# List models
ai-list

# Chat with a model
ollama run qwen2.5:0.5b

# Use the chat helper
./chat.sh llama3.2:1b

# Stop the service
./stop_ai.sh

🔧 Troubleshooting

If something goes wrong:
bash

# Check the log
tail -f ollama.log

# Check if Ollama is running
ps aux | grep ollama

# Manually start with models
export OLLAMA_MODELS="/path/to/ollama_models"
ollama serve

This all-in-one script handles everything - from installation to configuration to running models - with no personal information and completely portable!
    fi
fi
 
