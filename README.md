To address your request, we'll create a comprehensive Git Bash script that leverages `git-filter-repo` to construct your monorepo. This script will handle cloning, filtering, branch management, and provide clear instructions.

Here's a breakdown of the prompt and the subsequent script:

### **Prompt for Script Generation:**

```
Generate a Git Bash script for Windows (Bash v5 compatible) that performs the following monorepo creation steps using `git-filter-repo`:

1.  **Repository URLs:** The script should accept HTTP URLs for `new-repo`, `backend-repo`, and `frontend-repo` as arguments.
2.  **Clean Start:** Before starting, the script must remove any existing local clones of `new-repo`, `backend-repo`, and `frontend-repo` directories to ensure a clean execution every time.
3.  **Cloning:** Clone `new-repo`, `backend-repo`, and `frontend-repo` into temporary directories.
4.  **Folder Renaming Variables:** The script should define two variables, `BACKEND_TARGET_FOLDER` and `FRONTEND_TARGET_FOLDER`, which will represent the desired folder names for the backend and frontend code within the `new-repo` (e.g., `myapp-backend` and `myapp-frontend`). These variables should be easily configurable at the beginning of the script.
5.  **Branch Management:**
    * Read a list of branches from a file named `branches.txt`.
    * Assume `master` branch exists in both `backend-repo` and `frontend-repo`.
    * Assume all other branches listed in `branches.txt` are unique to `backend-repo` (i.e., they do not exist in `frontend-repo`).
    * For the `master` branch in `new-repo`:
        * It should contain the history and files from both `backend-repo/master` (placed in `BACKEND_TARGET_FOLDER`) and `frontend-repo/master` (placed in `FRONTEND_TARGET_FOLDER`).
        * Since `master` is protected and cannot be force-pushed, the script should prepare it for a normal push (which implies merging histories without conflicts that would require force-pushing).
    * For other branches (unique to `backend-repo`) in `new-repo`:
        * Each unique branch should contain the history and files from its corresponding `backend-repo` branch, placed in `BACKEND_TARGET_FOLDER`.
        * The script should ensure that these branches are created in `new-repo` with their respective histories, as if they originated solely from the backend.
6.  **`git-filter-repo` Usage:**
    * Apply `git-filter-repo` to `backend-repo` to move its contents into `BACKEND_TARGET_FOLDER`.
    * Apply `git-filter-repo` to `frontend-repo` to move its contents into `FRONTEND_TARGET_FOLDER`.
    * Perform a merge strategy to combine the filtered `backend-repo` and `frontend-repo` into `new-repo` on the `master` branch.
    * For the other branches, simply integrate the filtered backend branches into `new-repo`.
7.  **No Pushing:** The script *must not* push anything to the remote repositories. All operations should be local. The user will manually push after verification.
8.  **Logging and Commit Messages:** Include descriptive `echo` statements for progress tracking and informative commit messages where appropriate (especially for the initial monorepo commit on `master`).
9.  **Error Handling:** Include basic error handling (e.g., `set -e` to exit on error).
10. **Instructions:** Provide clear instructions on how to run the script, any prerequisites (like `git-filter-repo` installation), and what steps to take after the script completes successfully.
```

-----

### **The Script: `create_monorepo.sh`**

```bash
#!/bin/bash

# This script creates a monorepo from separate backend and frontend repositories
# using git-filter-repo. It handles cloning, filtering, and merging branches
# without pushing anything to remotes.

# --- Configuration Variables ---
BACKEND_TARGET_FOLDER="myapp-backend"   # Desired folder name for backend files in monorepo
FRONTEND_TARGET_FOLDER="myapp-frontend" # Desired folder name for frontend files in monorepo

# --- Prerequisites Check ---
echo "--- Checking Prerequisites ---"
if ! command -v git &> /dev/null
then
    echo "Error: Git is not installed. Please install Git and try again."
    exit 1
fi

if ! command -v git-filter-repo &> /dev/null
then
    echo "Error: git-filter-repo is not installed or not in your PATH."
    echo "Please install it (e.g., pip install git-filter-repo) and ensure it's accessible."
    exit 1
fi

if [ ! -f branches.txt ]; then
    echo "Error: branches.txt not found in the current directory."
    echo "Please create a branches.txt file with a list of branches, one per line."
    echo "Example content:"
    echo "master"
    echo "feature/new-api"
    echo "bugfix/login"
    exit 1
fi

# --- Input Validation ---
if [ "$#" -ne 3 ]; then
    echo "Usage: $0 <new-repo-http-url> <backend-repo-http-url> <frontend-repo-http-url>"
    echo "Example: $0 https://github.com/your/new-repo.git https://github.com/your/backend-repo.git https://github.com/your/frontend-repo.git"
    exit 1
fi

NEW_REPO_URL="$1"
BACKEND_REPO_URL="$2"
FRONTEND_REPO_URL="$3"

NEW_REPO_DIR="new-repo-temp"
BACKEND_REPO_DIR="backend-repo-temp"
FRONTEND_REPO_DIR="frontend-repo-temp"

# --- Clean Up Previous Runs ---
echo "--- Cleaning up previous temporary directories (if any) ---"
rm -rf "$NEW_REPO_DIR" "$BACKEND_REPO_DIR" "$FRONTEND_REPO_DIR"

# --- Clone Repositories ---
echo "--- Cloning repositories ---"
git clone "$NEW_REPO_URL" "$NEW_REPO_DIR" || { echo "Failed to clone new-repo. Exiting."; exit 1; }
git clone "$BACKEND_REPO_URL" "$BACKEND_REPO_DIR" || { echo "Failed to clone backend-repo. Exiting."; exit 1; }
git clone "$FRONTEND_REPO_URL" "$FRONTEND_REPO_DIR" || { echo "Failed to clone frontend-repo. Exiting."; exit 1; }

# --- Process Backend Repository ---
echo "--- Processing backend repository: $BACKEND_REPO_DIR ---"
cd "$BACKEND_REPO_DIR" || { echo "Failed to enter backend-repo-temp. Exiting."; exit 1; }
echo "Applying git-filter-repo to move backend files into '$BACKEND_TARGET_FOLDER'..."
git filter-repo --to-subdirectory-filter "$BACKEND_TARGET_FOLDER" --force || { echo "git-filter-repo failed for backend. Exiting."; exit 1; }
echo "Backend repository filtered successfully."
cd .. # Go back to original directory

# --- Process Frontend Repository ---
echo "--- Processing frontend repository: $FRONTEND_REPO_DIR ---"
cd "$FRONTEND_REPO_DIR" || { echo "Failed to enter frontend-repo-temp. Exiting."; exit 1; }
echo "Applying git-filter-repo to move frontend files into '$FRONTEND_TARGET_FOLDER'..."
git filter-repo --to-subdirectory-filter "$FRONTEND_TARGET_FOLDER" --force || { echo "git-filter-repo failed for frontend. Exiting."; exit 1; }
echo "Frontend repository filtered successfully."
cd .. # Go back to original directory

# --- Integrate into New Monorepo ---
echo "--- Integrating filtered repositories into '$NEW_REPO_DIR' ---"
cd "$NEW_REPO_DIR" || { echo "Failed to enter new-repo-temp. Exiting."; exit 1; }

# --- Configure Git for new-repo (important for merges) ---
git config user.email "monorepo_migration@example.com"
git config user.name "Monorepo Migration Script"

# Fetch all branches from filtered backend and frontend repos
echo "Fetching all branches from filtered backend and frontend repositories..."
git remote add backend_filtered ../"$BACKEND_REPO_DIR"
git remote add frontend_filtered ../"$FRONTEND_REPO_DIR"
git fetch backend_filtered || { echo "Failed to fetch from backend_filtered. Exiting."; exit 1; }
git fetch frontend_filtered || { echo "Failed to fetch from frontend_filtered. Exiting."; exit 1; }

# --- Handle Master Branch ---
echo "--- Processing 'master' branch ---"
git checkout master || { echo "Failed to checkout master in new-repo. Exiting."; exit 1; }

# Merge frontend master first
echo "Merging frontend/master into new-repo/master..."
if git merge --allow-unrelated-histories frontend_filtered/master -m "feat: Initial merge of frontend into monorepo"; then
    echo "Frontend master merged successfully."
else
    echo "Warning: Frontend master merge had conflicts. Please resolve manually if needed after script completion."
fi

# Merge backend master second
echo "Merging backend/master into new-repo/master..."
if git merge --allow-unrelated-histories backend_filtered/master -m "feat: Initial merge of backend into monorepo"; then
    echo "Backend master merged successfully."
else
    echo "Warning: Backend master merge had conflicts. Please resolve manually if needed after script completion."
fi

# --- Handle Other Branches from branches.txt ---
echo "--- Processing other branches from branches.txt ---"
# Read branches line by line, skipping master as it's handled separately
grep -v '^master$' ../branches.txt | while IFS= read -r branch; do
    if [ -z "$branch" ]; then
        continue # Skip empty lines
    fi
    echo "Creating and populating branch '$branch' from backend repository..."
    if git checkout -b "$branch" backend_filtered/"$branch"; then
        echo "Branch '$branch' created and populated successfully."
    else
        echo "Warning: Could not create branch '$branch' from backend_filtered. It might not exist in the backend repo or there was an issue."
        # If the branch couldn't be created directly, it means it didn't exist in backend_filtered.
        # In this scenario, we might want to skip it or log it more prominently.
        # For this script's assumption (branches are unique to backend), this "failure" is a warning.
    fi
    git checkout master # Switch back to master to continue creating branches
done

echo "--- Monorepo creation process completed locally! ---"
echo "You are currently in the '$NEW_REPO_DIR' directory."
echo "Please review the changes and verify the repository structure."
echo "No changes have been pushed to your remote repositories yet."

# --- Next Steps for User ---
echo "--- Next Steps: ---"
echo "1. Verify the contents of '$NEW_REPO_DIR':"
echo "   - Check the '$BACKEND_TARGET_FOLDER' and '$FRONTEND_TARGET_FOLDER' directories."
echo "   - Use 'git log' to inspect commit histories on 'master' and other branches."
echo "   - Use 'git branch -a' to see all local branches."
echo "2. If everything looks correct, navigate into '$NEW_REPO_DIR':"
echo "   cd $NEW_REPO_DIR"
echo "3. Push your changes to the remote 'new-repo':"
echo "   git push --all origin"
echo "   git push --tags origin (if you have tags you want to push)"
echo "   Note: For the master branch, a normal push will work since we merged histories."
echo "         For other branches, it will be a normal push of new branches."
echo "4. Consider setting up a new default branch on your remote new-repo if needed."
```

### **Instructions on Running the Script:**

1.  **Save the Script:** Save the content above into a file named `create_monorepo.sh` (or any other `.sh` extension) in your desired directory.
2.  **Create `branches.txt`:** In the **same directory** as `create_monorepo.sh`, create a file named `branches.txt`.
      * List all branches you want to transfer, one per line.
      * Ensure `master` (or your primary branch name) is included.
      * Example `branches.txt`:
        ```
        master
        feature/auth
        bugfix/payments
        ```
3.  **Make Executable (Optional on Windows, but good practice):**
    Open Git Bash, navigate to the directory where you saved the script, and run:
    ```bash
    chmod +x create_monorepo.sh
    ```
4.  **Run the Script:** Execute the script from Git Bash, providing the HTTP URLs for your repositories as arguments.
    ```bash
    ./create_monorepo.sh https://github.com/your-org/new-repo.git https://github.com/your-org/backend-repo.git https://github.com/your-org/frontend-repo.git
    ```
    *Replace the example URLs with your actual repository URLs.*

### **Prerequisites:**

1.  **Git:** Ensure Git is installed and available in your PATH. You can download it from [git-scm.com](https://git-scm.com/).
2.  **`git-filter-repo`:** This tool is crucial for rewriting repository histories.
      * **Installation:**
        ```bash
        pip install git-filter-repo
        ```
        (You'll need Python and `pip` installed for this. If not, follow instructions on `git-filter-repo`'s GitHub page for alternative installation methods.)
      * Ensure `git-filter-repo` is in your system's PATH.

### **After Successful Script Execution:**

The script will create three temporary directories in the same location where you ran the script: `new-repo-temp`, `backend-repo-temp`, and `frontend-repo-temp`. The `new-repo-temp` directory will contain your newly constructed monorepo.

The script explicitly states: "**No changes have been pushed to your remote repositories yet.**"

Here are the critical next steps:

1.  **Verification:**

      * Navigate into the `new-repo-temp` directory: `cd new-repo-temp`
      * **Inspect folder structure:** Verify that `myapp-backend` and `myapp-frontend` (or whatever names you set for `BACKEND_TARGET_FOLDER` and `FRONTEND_TARGET_FOLDER`) exist and contain the correct files from their respective original repositories.
      * **Check `master` branch:**
          * `git checkout master`
          * `git log --oneline --graph` to see the merged history.
          * Confirm files from both `myapp-backend` and `myapp-frontend` are present.
      * **Check other branches:**
          * For each unique branch listed in `branches.txt` (e.g., `git checkout feature/auth`), verify that only the `myapp-backend` folder exists and contains the correct files with the correct history for that branch.
          * Use `git branch -a` to see all local branches that were created.

2.  **Push to Remote (`new-repo`):**

      * Once you are satisfied with the local `new-repo-temp` setup, you can push the changes to your actual remote `new-repo`.
      * **From within the `new-repo-temp` directory:**
        ```bash
        git push --all origin
        git push --tags origin # If you also want to push any tags
        ```
      * Since the `master` branch was created by merging histories, a normal `git push` should work without needing `--force`. The other branches are essentially new branches being pushed.

3.  **Clean Up (Optional after Push):**
    After successfully pushing to your remote `new-repo`, you can delete the temporary local directories:

    ```bash
    cd .. # Go back to the directory where create_monorepo.sh is
    rm -rf new-repo-temp backend-repo-temp frontend-repo-temp
    ```

This detailed approach ensures a robust and verifiable monorepo creation process.
