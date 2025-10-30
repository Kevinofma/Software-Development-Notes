# Getting Started

## Starting a Project

This section will help you start/initialize a project. The following is covered:

- Set up a basic Python development container in VS Code.
- Initialize and configure a GitHub repository for a static site.
- Deploy your site to GitHub Pages with GitHub Actions for CI/CD.

### Prerequisites

Ensure you have the following:

- A GitHub account: If you don’t have one yet, sign up at [GitHub](https://github.com/).
- Git installed: [Install Git](https://git-scm.com/) if you don’t already have it.
- Visual Studio Code (VS Code): Download and install it from [here](https://code.visualstudio.com/download).
- Docker: Required to run the dev container. Get Docker [here](https://www.docker.com/products/docker-desktop/).

---

## Part 1. Project Setup: Creating the Repository

### Step 1. Create a Local Directory and Initialize Git

(A) Open your terminal or command prompt.  

(B) Create a new directory for your project and change into it:

??? note
    You can do this in the terminal using the following commands:

    ```bash
    mkdir project-name
    cd project-name
    ```

    Make sure you `cd` into the folder/path you want to make the project first.

    You can also just simply make this either using your file manager or in VS Code.

(C) Initalize a new Git Repository:

```bash
git init
```

This creates a .git directory and starts tracking your project with Git.

---

### Step 2. Create a Remote Repository on GitHub

1. Log in to your [GitHub account](https://github.com/) and navigate to the **[Create a New Repository](https://github.com/new)** page.  
2. Fill in the details as follows:  
   - **Repository Name**: `project-name-on-github`  
   - **Description**: "project description"  
   - **Visibility**: Public  
3. **Do not** initialize the repository with a README, `.gitignore`, or license.  
4. Click **Create Repository**.  

---

### Step 3. Link Your Local Repository to GitHub

1. Add the GitHub repository as a remote:  

    ```bash
    git remote add origin https://github.com/<your-username>/project-name-on-github.git
    ```

    !!! note
        Replace `<your-username>` with your actual GitHub username.  

2. Check your default branch name with:  

    ```bash
    git branch
    ```

    If it’s not `main`, rename it:  

    ```bash
    git branch -M main
    ```

    !!! info
        Older versions of Git use `master` as the default branch name, but the current standard is `main`.  

3. Push your local commits to GitHub:  

    ```bash
    git push --set-upstream origin main
    ```

    ??? tip "Understanding the `--set-upstream` flag"
        The command `git push --set-upstream origin main` pushes your local `main` branch to the remote `origin`.  
        The `--set-upstream` (or `-u`) flag links your local `main` branch to the remote one, so in the future you can simply run:  

        ```bash
        git push
        git pull
        ```  

        without specifying the branch each time.  

4. Back in your web browser, refresh your GitHub repository.  
   - You should now see your initial commit on GitHub.  
   - To double-check locally, run:  

     ```bash
     git log
     ```

     The commit ID and message should match what you see on GitHub.  

---

## Part 2. Setting Up the Development Environment
### Step 1. Add a Development Container Configuration

1. In VS Code, open your project directory. You can do this via: **File > Open Folder**.  
2. Install the **Dev Containers** extension for VS Code.  
3. Create a `.devcontainer` directory in the root of your project. Inside this "hidden" configuration directory, create a file called:  

    ```
    .devcontainer/devcontainer.json
    ```

4. The `devcontainer.json` file defines the configuration for your development environment. Here’s an example template:  

    ```json
    {
      "name": "My Project Dev Container",
      "image": "mcr.microsoft.com/devcontainers/python:latest",
      "customizations": {
        "vscode": {
          "settings": {},
          "extensions": ["ms-python.python"]
        }
      },
      "postCreateCommand": "pip install -r requirements.txt"
    }
    ```

    ??? note "Explanation of each field"
    
        * **name**: A descriptive name for your dev container.
        
        * **image**: The Docker image to use (here, a Python environment). Microsoft maintains base images for many languages, or you can create your own.
        
        * **customizations**: Add useful VS Code configurations, like installing extensions automatically for anyone using this container.
        
        * **postCreateCommand**: Commands to run after the container is created. For example, install dependencies from requirements.txt.

        ??? note "requirements.txt"

            This is a file created at the **root of your project directory** called `requirements.txt`.  
            It lists the Python packages that should be installed in the container for the project to run.  

            For example, this documentation project’s `requirements.txt` contains:  

            ```txt
            mkdocs-material~=9.5
            ```




5. After saving the file, you can reopen your project in the dev container via the VS Code command palette: **Remote-Containers: Reopen in Container**.

---

## Part 3. Deploying with GitHub Pages (or any static site)

In this section, you'll set up an automated deployment process using **GitHub Actions**, a tool for Continuous Integration and Continuous Deployment (CI/CD). This means every time you push changes to GitHub, a series of automated steps will run to build, test, and deploy your site or project. This ensures your site is always up-to-date with your latest changes.

### What You'll Do

- Add a GitHub Actions workflow to define the automated deployment process.
- Push changes and observe the deployment process in action.

---

### Step 1. Add a GitHub Action for CI/CD

Create a workflow file in your repository:

```bash
mkdir -p .github/workflows
code .github/workflows/ci.yml
```

This file defines the steps for your CI/CD workflow.

```yaml
name: Deploy Project

on:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.x'
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Build and Deploy
        run: <your-deploy-command>
```

??? note "Explanation of workflow fields"
    **name**: This is the name of your workflow. It will appear in the GitHub Actions UI so you can easily identify it (e.g., "Deploy Project").

    **on.push.branches**: Specifies that the workflow should run whenever changes are pushed to the listed branches (here, the `main` branch). You could add more branches if needed.

    **permissions.contents**: Grants the workflow permission to write back to your repository. This is required if your deployment pushes to a branch like `gh-pages`.

    **jobs**: Defines a set of tasks that will run as part of the workflow. You can have multiple jobs if needed.

    **deploy**: This is the ID of the job. It can be any name, and is referenced in logs or for dependencies between jobs.

    **runs-on**: Specifies the environment the job runs on. `ubuntu-latest` means GitHub will provide the latest Ubuntu runner. Other options include `windows-latest` or `macos-latest`.

    **steps**: Lists individual steps in the job. Each step is executed in order.

    **uses**: Specifies a prebuilt action from GitHub Marketplace or a repository. 

    * Example: `actions/checkout@v4` checks out your repository so the workflow can access the code.  
    * Example: `actions/setup-python@v5` sets up a Python environment on the runner.

    **with**: Provides parameters to an action. For example, `python-version: '3.x'` tells the Python setup action which Python version to install.

    **name** (inside steps): Optional name for the step. This shows in the Actions UI logs for clarity, e.g., "Install dependencies".

    **run**: Specifies a command to execute on the runner. You can write any shell command here, such as installing dependencies or deploying your site.


---

### Step 2. Push Changes and Test the Workflow

1. Add and commit your workflow file and any other project changes:

```bash
git add .
git commit -m "Set up CI/CD workflow"
```

---

### Understanding Your CI/CD Workflow

Once set up, the workflow will run automatically whenever you push changes to the repository:

1. Commit Changes Locally: Edit and save your project files, then commit the changes.

2. Push to GitHub: The push triggers the GitHub Action workflow.

3. Workflow Executes:

    * Checks out the repository

    * Sets up the runtime environment

    * Installs dependencies

    * Builds the project

    * Deploys to the specified branch or server

4. Site/Project Updated: The latest changes are now live automatically, without manual deployment.

This "push-to-deploy" workflow is widely used in professional development environments.