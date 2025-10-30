# Introduction to CI/CD

Modern software development moves fast. Teams need to deliver features quickly, fix bugs rapidly, and maintain high-quality code—all while ensuring deployments are smooth and reliable. This is where **CI/CD** comes in.

-   **Continuous Integration (CI)** automates the process of running tests on new code changes whenever they are pushed to a central repository. This ensures that issues are caught early and developers get fast feedback.
-   **Continuous Deployment (CD)** takes things a step further by **automating the deployment of every successfully tested change directly into production**—without manual intervention. This allows teams to ship updates multiple times a day with confidence.

!!! note "Continuous Delivery vs. Continuous Deployment"
    **Continuous Delivery** ensures that every tested change is ready for deployment but still requires a manual approval step before going live. **Continuous Deployment**, on the other hand, removes this manual step and automatically pushes changes to production once they pass all verification checks.

    In this tutorial, we’re focusing on **Continuous Deployment**, where code moves to production immediately after passing CI tests.

## Why CI/CD Matters

Manually running tests and deploying software can be slow, inconsistent, and prone to human error. CI/CD automates these critical steps, leading to:

-   **Faster development cycles** – Developers can push changes more frequently without worrying about breaking things.
-   **Less anxiety** – Automated testing and deployment remove the guesswork, making releases predictable and repeatable.
-   **Higher confidence** – If every change is tested and verified before it reaches production, teams can move forward with fewer worries about stability.

By embracing automation, teams shift from a culture of hesitation and uncertainty to one of confidence and speed.

---

## CI/CD Pipeline Tutorial with the RPS Game

In this tutorial, you’ll configure a full **CI/CD pipeline** for our Rock, Paper, Scissors (RPS) application. We'll use **GitHub Actions for CI** and a Kubernetes-based platform **(OKD) for CD**.

### The Application Code

We will use the same Rock, Paper, Scissors application we built in the "Service Layer and Dependency Injection" and "Introduction to Testing" sections. The code for `models.py`, `services.py`, and `main.py` will be the same.

#### Dependencies (`requirements.txt`)

First, ensure you have a `requirements.txt` file that includes all the necessary packages for the application and for testing.

```text title="requirements.txt"
fastapi[standard]~=0.115.7
pytest~=8.3.4
```

#### The Tests

Our CI pipeline will run the tests we wrote previously. These include unit tests for the service and integration tests for our API endpoints. Having these tests is crucial for an effective CI process.

---

### Part 1: Continuous Integration (CI) with GitHub Actions

Continuous Integration is controlled by a GitHub Action workflow file. This file tells GitHub what to do whenever code is pushed to the repository.

#### The Workflow File (`.github/workflows/test.yml`)

Create a file at this path in your repository. This workflow will check out your code, install dependencies, and run `pytest`.

```yaml title=".github/workflows/test.yml"
name: Run Python Tests

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.x"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Test with pytest (CI)
        run: pytest

      - name: Deploy to production (CD)
        if: success()
        run: |
          curl -X POST "${{ secrets.WEBHOOK_URL }}"
```

??? question "What does the `curl` command do?"
    The `Deploy to production (CD)` step is the bridge between your CI and CD systems.
    `yaml run: | curl -X POST "${{ secrets.WEBHOOK_URL }}" `

    * `curl`: This is a command-line tool for making web requests.
    * `-X POST`: This specifies that we are sending an HTTP `POST` request. A `POST` request is used here because we are asking the server (OKD) to take an action—in this case, start a new deployment.
    * `"${{ secrets.WEBHOOK_URL }}"`: This is the most important part.
    * `secrets.WEBHOOK_URL`: This securely accesses the secret value you stored in your GitHub repository's settings. You should never paste secret URLs or tokens directly into your code.
    * `${{ ... }}`: This is the syntax GitHub Actions uses to substitute the value of a variable or secret into the workflow.

    In short, this command sends a secure notification to your deployment server (OKD), telling it that the tests have passed and it's time to start building and deploying the new version of the application.

Once you commit and push this file, go to the "Actions" tab in your GitHub repository. You should see the workflow running. If all tests pass, you'll see a green checkmark\! This confirms that your CI pipeline is working. ✅

-----

### Part 2: Continuous Deployment (CD) with OKD

Now that our tests pass, we'll set up a pipeline to automatically deploy the application to a production environment (OKD).

#### 1. The Dockerfile

To deploy our app, we need to package it into a container. A `Dockerfile` provides the instructions to do this.

```dockerfile title="Dockerfile"
# Use the official Python 3.13 image as the base image.
FROM python:3.13

# Set the working directory in the container.
WORKDIR /app

# Copy requirements file to the container.
COPY ./requirements.txt /app/requirements.txt

# Install the Python dependencies.
RUN pip install --upgrade pip && \
    pip install -r requirements.txt

# Copy the rest of the application code.
COPY . /app

# Expose port 8080 which uvicorn will run on.
EXPOSE 8080

# Command to run FastAPI in production mode using uvicorn.
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

??? info "Breaking Down the Dockerfile"
    Each line in a Dockerfile is a specific instruction for building your application's container image.

    - **`FROM python:3.13`**: This sets the foundation. It tells Docker to start with an official base image that already has Python 3.13 installed.
    - **`WORKDIR /app`**: This sets the default working directory inside the container to `/app`. All subsequent commands (`COPY`, `RUN`, `CMD`) will be executed from this directory.
    - **`COPY ./requirements.txt /app/requirements.txt`**: This copies just the `requirements.txt` file from your local project into the container. We do this first so Docker can cache the installed dependencies. If your code changes but `requirements.txt` doesn't, Docker can skip reinstalling everything, making builds faster.
    - **`RUN pip install ...`**: This command executes inside the container. It installs the Python packages listed in `requirements.txt`.
    - **`COPY . /app`**: This copies the rest of your application code (like `main.py`, `services.py`, etc.) into the `/app` directory inside the container.
    - **`EXPOSE 8080`**: This is informational. It tells Docker that the application inside the container will listen on port 8080. It doesn't actually open the port, but serves as documentation for the person running the container.
    - **`CMD ["uvicorn", ...]`**: This specifies the default command to run when the container starts. It starts the `uvicorn` server to run your FastAPI application, making it accessible on all network interfaces (`--host 0.0.0.0`) on port 8080 (`--port 8080`).

#### 2. Generate a GitHub Personal Access Token (PAT)

Your deployment platform (OKD) needs permission to pull your code from your private GitHub repository. To grant this, you'll create a Personal Access Token.

1.  **In GitHub**, go to **Settings** \> **Developer settings** \> **Personal access tokens** \> **Tokens (classic)**.
2.  Click **Generate new token (classic)**.
3.  Give it a descriptive name (e.g., "OKD-Deployment-Key").
4.  Select the **`repo`** scope.
5.  Click **Generate token** and **copy the token immediately**. You won't be able to see it again.

#### 3. Configure the Deployment Environment

(These steps assume you have access to an OKD cluster and have the `oc` command-line tool installed).

??? info "For UNC Students"
    Setting up deployment takes some more effort because we need to stand up a production cloud environment. The production environment will be UNC Cloud Apps' "OKD" cluster.

    You need to be connected to Eduroam, or connected to UNC VPN (instructions here), in order to successfully use OKD. If you are on a home network, UNC guest, or other network, be sure to connect via VPN.

    Login to OKD by going to: <https://console.apps.unc.edu>

    (If your login does not succeed, it's likely because you did not previously register for Cloud Apps. You can do so by going to <https://cloudapps.unc.edu/> and following the Sign Up steps. It can take up to 15 minutes following Sign Up for the OKD link above to work correctly. In the interim, feel free to follow along with your neighbor.)

    Once logged in you should see OKD in the upper-left corner. If you see "Red Hat", be sure you opened the link above.

    Now that you are logged in, go to the upper-right corner and click your `ONYEN` and go to the `"Copy Login Command"` link. Click `Display Token`. In this, copy the command in the first text box. Paste it into your dev container's terminal (which has the oc command-line application for interfacing with a Red Hat Open-Shift Kubernetes cluster installed).

    Before proceeding, switch to your personal OKD project using your ONYEN. For example, if your ONYEN is "jdoe", run:

    ```bash
    oc project comp590-140-25sp-jdoe
    ```

    If the above command fails, restart the steps above! The following will not work until you are able to access your project via oc.

1.  **Log in to OKD** and select your project.
2.  **Create a secret** to store your GitHub credentials. Replace `<your-github-username>` and `<your-github-pat>` with your actual username and the token you just generated.
    ```bash
    # Choose a name for your secret, e.g., "github-repo-secret"
    oc create secret generic <your-secret-name> \
        --from-literal=username=<your-github-username> \
        --from-literal=password=<your-github-pat> \
        --type=kubernetes.io/basic-auth
    ```
3.  **Create the application** in OKD. This command tells OKD to build and deploy your application from your Git repository using the `Dockerfile`.
    ```bash
    # Choose a name for your application, e.g., "my-rps-game"
    oc new-app . \
    --name=<your-app-name> \
    --source-secret=<your-secret-name> \
    --strategy=docker \
    --labels=app=<your-app-name>
    ```
    * `--name`: This will be the name of your application, service, and route in OKD.

    * `--source-secret`: This must match the secret name you created in the previous step.

    * `--labels=app=<your-app-name>`: This adds a label to all the resources created (`Deployment`, `Service`, etc.). This is extremely useful because it allows you to manage all related components with a single command, like when you want to delete the application later.

4.  **Expose the service** to make it accessible on the internet.
    ```bash
    oc create route edge --service=<your-app-name>
    ```
    You can get the public URL by running oc get route <your-app-name>.

#### 4. Set up the Webhook for Automatic Deployments

The final step is to connect our CI pipeline to our CD platform. We'll use a webhook: after the tests pass, GitHub Actions will send a request to a special URL in OKD, triggering a new deployment.

1.  **Get the webhook URL** from your OKD build configuration:
    ```bash
    oc describe bc/<your-app-name> | grep -C 1 generic
    ```
2.  **In your GitHub repository**, go to **Settings \> Secrets and variables \> Actions**.
3.  Click **New repository secret**.
4.  Name the secret `WEBHOOK_URL` and paste the full webhook URL from the previous step.
5.  The `curl` command in our `test.yml` workflow file will now automatically trigger a new deployment on OKD every time the tests successfully pass on the `main` branch\!

And that's it! You now have a fully automated CI/CD pipeline. Every time you push a change, it will be tested, and if successful, deployed directly to production.
