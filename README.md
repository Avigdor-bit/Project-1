# Project-1

Hello everyone...
how are you
Hope it's good..
carrier:
The purpose of the project
How to create integrate Jenkins with GitHub Using SSH Credentials
on Ubuntu 26.0

below
instructions

Here is a comprehensive, step-by-step guide to integrating Jenkins with GitHub using SSH credentials for secure, automated builds.

### 🛠️ Prerequisites

Before you begin, ensure you have the following:

1. **A GitHub Account**: You'll need access to your GitHub repository settings.
2. **A Jenkins Server**: Jenkins should be installed and running. This can be on your local machine, a server, or in a Docker container.
3. **Git Installed on Jenkins Server**: Jenkins needs Git to clone repositories. You can check with `git --version` and install it if necessary (e.g., `sudo apt install git` on Ubuntu).
4. **Jenkins Git Plugin**: The `Git Plugin` and `Git Client Plugin` are essential and usually come pre-installed with Jenkins. You can check this in `Manage Jenkins` > `Plugins` > `Installed`.

---

### 🔑 Step 1: Generate an SSH Key Pair

First, you need to create a dedicated SSH key pair on the Jenkins server. This key will be used for secure communication with GitHub.

1. Access the terminal of your Jenkins server.
2. Run the following command. For modern security, the `ed25519` algorithm is recommended, but `rsa` is also common. 
```bash 
# For ED25519 (recommended) 
ssh-keygen -t ed25519 -C "jenkins@your-server" 

# OR for RSA 
ssh-keygen -m PEM -t rsa -P "" -C "jenkins@your-server" 
```
3. When prompted to "Enter a file in which to save the key," press `Enter` to accept the default location (`~/.ssh/id_ed25519` or `~/.ssh/id_rsa`).
4. When prompted for a passphrase, press Enter twice to leave it empty. Jenkins needs to use this key non-interactively, so a passphrase would cause authentication to fail.

This process creates two files: a private key (`id_ed25519`) and a public key (`id_ed25519.pub`).

---

### ☁️ Step 2: Add the SSH Public Key to GitHub

Now, you need to link the public key to your GitHub account so Jenkins can access your repositories.

1. Display the contents of the **public key** file. 
```bash 
cat ~/.ssh/id_ed25519.pub 
```
2. Copy the entire output to your clipboard.
3. Go to GitHub, click on your profile photo (top right), and go to **Settings** > **SSH and GPG keys**.
4. Click **New SSH Key**.
5. Provide a descriptive **Title** (e.g., "Jenkins Server Key").
6. Paste the public key into the **Key** field and click **Add SSH Key**.

---

### 🧑‍💻 Step 3: Configure SSH Credentials in Jenkins

Next, you'll store the **private key** securely in Jenkins as a credential.

1. Open your Jenkins dashboard in a browser.
2. Go to **Manage Jenkins** > **Credentials** > **System** > **Global credentials (unrestricted)** > **Add Credentials**.
3. From the **Kind** dropdown, select **SSH Username with private key**.
4. Fill in the details: 
* **ID**: Give it a meaningful, unique ID (e.g., `github-ssh-key`). This will be used to reference the credentials later. 
* **Description**: A short description (e.g., "SSH Key for GitHub"). 
* **Username**: Enter your GitHub username. 
* **Private Key**: Choose **Enter directly**. Paste the entire contents of your **private key** file (`id_ed25519` or `id_rsa`) into the text box. Ensure you include the `-----BEGIN OPENSSH PRIVATE KEY-----` (or similar) header and footer.
5. Click **Create**.

---

### ✅ Step 4: (Optional) Verify the SSH Connection

For peace of mind, it's a good practice to test the connection from the Jenkins server to GitHub.

1. On the Jenkins server, run the following command to add `github.com` to the server's `known_hosts` file and test the connection. 
```bash 
ssh -T git@github.com 
```
2. You should see a success message similar to: ``Hi <your-username>! You've successfully authenticated...`.

---

### 🔗 Step 5: Configure Your Jenkins Project

Now you can configure your Jenkins project (Freestyle or Pipeline) to use the SSH key for cloning.

#### **For a Freestyle Project**:

1. Create a new item or open an existing Freestyle project.
2. In the **Source Code Management** section, select **Git**.
3. In the **Repository URL** field, use the SSH URL from your GitHub repository. It must start with `git@`: 
``` 
git@github.com:your-username/your-repo-name.git 
```
4. From the **Credentials** dropdown, select the SSH credential you created in Step 3 (`github-ssh-key`).
5. Specify the **Branches to build** (e.g., `*/main` or `*/master`).
6. Save your configuration.

#### **For a Pipeline (Jenkinsfile):**

You can use the `sshagent` step from the **SSH Agent Plugin** to securely use the credentials within your pipeline.

```groovy
pipeline { 
agent any 

stages { 
stage('Checkout') { 
steps { 
// 'github-ssh-key' is the ID of the credential you created 
sshagent(credentials: ['github-ssh-key']) { 
git branch: 'main',

]
EOF
