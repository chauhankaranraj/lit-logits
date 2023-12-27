# Working Environment

This doc walks (for the most part, future me) through setting up a data science / machine learning environment.

## Zsh

1. Install zsh.
    ```
    sudo apt update
    sudo apt install zsh -y
    ```

2. Configure zsh.
    ```
    zsh
    ```

3. Set zsh as the default shell.
    ```
    sudo chsh -s $(which zsh)
    sudo chsh -s $(which zsh) ubuntu
    ```

## Oh My Zsh

1. Install Oh My Zsh.
    ```
    sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
    ```

2. Install zsh-autosuggestions plugin.
    ```
    git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
    # Next, add zsh-autosuggestions to plugins=() section in ~/.zshrc
    ```

3. Install zsh-syntax-highlighting plugin.
    ```
    git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
    # Next, add zsh-syntax-highlighting to plugins=() section in ~/.zshrc
    ```

4. Set theme
    ```
    vim ~/.zshrc
    # set ZSH_THEME=af-magic
    ```

5. Add any frequently used aliases.
    ```
    vim ~/.zshrc
    # Add the following under the section titled "personal aliases"
    alias cls="clear"
    alias dk="docker"
    alias pe="poetry"
    ```

## Dependency Management

Python can sometimes be difficult to tame due to its dependency and packaging quirks. In my (little) experience, I've found that a combination of `pyenv` and `poetry` is able to resolve most python dependency issues well. That said, I also like to keep conda installed for the rare occasions that the former fails.

### Miniconda

1. Install miniconda using the instructions on the [official docs](https://docs.conda.io/projects/miniconda/en/latest/).
    ```
    mkdir -p ~/miniconda3
    wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
    bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
    rm -rf ~/miniconda3/miniconda.sh
    ```

2. Init conda for zsh (this will add content to `.zshrc`) and disable activation of the base environment by default.
    ```
    ~/miniconda3/bin/conda init zsh
    ~/miniconda3/bin/conda config --set auto_activate_base false
    ```

### Pyenv

1. Build the pre-requisite system dependencies.
    ```
    sudo apt update; sudo apt install -y build-essential libssl-dev zlib1g-dev \
    libbz2-dev libreadline-dev libsqlite3-dev curl \
    libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev
    ```

2. Install pyenv using the quick and dirty way.
    ```
    curl https://pyenv.run | bash
    ```

3. Add the following pyenv initialization lines to `.zshrc`
    ```
    # >>> pyenv init >>>
    export PYENV_ROOT="$HOME/.pyenv"
    [[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
    eval "$(pyenv init -)"
    # <<< pyenv init <<<
    ```

4. Install the versions of Python you intend to use for your projects. NOTE: I _highly_ advice against using system python in any of your projects, even if it's the exact version you want. Just use pyenv installed Python binaries for isolated and clean environments.
    ```
    pyenv install 3.10.13
    pyenv install 3.8.18
    ```

### Poetry

1. Install using the quick and dirty way shown on the [official docs](https://python-poetry.org/docs/#installing-with-the-official-installer). NOTE: use system python for this, not pyenv python binaries.
    ```
    curl -sSL https://install.python-poetry.org | python3 -
    ```

2. Update `PATH` variable by adding the following to `~/.zshrc`.
    ```
    # >>> poetry init >>>
    export PATH="/home/ubuntu/.local/bin:$PATH"
    # <<< poetry init <<<
    ```

3. (Optional) Configure poetry to create the virtualenv for a project within its own directory.
    ```
    poetry config virtualenvs.in-project true
    ```

## Working with pyenv + poetry

The most reliable and reproducible way to work on Python projects using the above tools is as follows.

1. Create a directory for the project, for example
    ```
    mkdir py-3-8-18
    cd py-3-8-18
    ```

2. If the Python version (let's say, `3.8.18`) you intend to use for this project is not already installed by pyenv, then do so first.
    ```
    pyenv install 3.8.18
    ```

3. Set the local Python version in this directory to be the intended version.
    ```
    pyenv local 3.8.18
    python -V    # should output 3.8.18
    ```

4. Initialize poetry for this project. In the interactive prompts that follow, set Python version accordingly (`3.8.18` in this example) and optionally list the dependencies of the project.
    ```
    poetry init
    ```

5. Activate the virtual environment and then install the dependencies. Once this is done, you can work on the project scripts as you normally would.
    ```
    poetry shell
    poetry install
    ```

6. After you're done, quit the virtual environment as follows
    ```
    exit
    ```
