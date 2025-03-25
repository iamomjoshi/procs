# GitHub Command Line Tutorial (Linux Only)

## 1. Installing Git on Linux
Git is available in most package managers. Install it using:

- **Debian/Ubuntu-based systems:**
  ```sh
  sudo apt update && sudo apt install git -y
  ```
- **Arch-based systems:**
  ```sh
  sudo pacman -S git
  ```
- **Fedora-based systems:**
  ```sh
  sudo dnf install git
  ```
- **OpenSUSE:**
  ```sh
  sudo zypper install git
  ```

Check installation:
```sh
git --version
```

## 2. Configuring Git
Before using Git, set up your username and email:
```sh
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```
To verify configuration:
```sh
git config --list
```

## 3. Initializing a Repository
To start a new repository:
```sh
git init
```
This creates a hidden `.git` directory for version control.

## 4. Cloning a Repository
To clone an existing repository:
```sh
git clone <repository_url>
```
For SSH:
```sh
git clone git@github.com:username/repository.git
```

## 5. Adding and Committing Changes
```sh
git add <file>        # Add specific file
git add .            # Add all modified files
git commit -m "Commit message"
```
To edit the last commit message:
```sh
git commit --amend
```

## 6. Pushing Changes
```sh
git push origin main  # Push changes to GitHub
```
For the first push:
```sh
git push -u origin main
```

## 7. Pulling Changes from Remote Repository
```sh
git pull origin main
```
If there are conflicts:
```sh
git merge --abort  # Abort merge if needed
```

## 8. Creating and Switching Branches
```sh
git branch new-branch  # Create a new branch
git checkout new-branch  # Switch to new branch
git checkout -b new-branch  # Create and switch
```

## 9. Merging Branches
```sh
git checkout main  # Switch to main branch
git merge new-branch  # Merge changes
git branch -d new-branch  # Delete merged branch
```

## 10. Resetting and Reverting Changes
```sh
git reset --hard HEAD  # Undo all uncommitted changes
git revert <commit_hash>  # Create a new commit that undoes a previous one
```

## 11. Deleting Files in Git
```sh
git rm -r <file_or_folder>
git commit -m "Deleted file"
git push origin main
```

## 12. Deleting a Remote Repository (DANGER)
```sh
git push --force origin --delete main
```

## 13. Checking Git Status and Logs
```sh
git status  # Show current status
git log --oneline --graph --decorate --all  # View commit history
```

## 14. Stashing Changes (Temporary Save)
```sh
git stash  # Save uncommitted changes temporarily
git stash pop  # Apply stashed changes
git stash list  # View all stashes
git stash drop stash@{0}  # Delete a specific stash
```

## 15. Resolving Merge Conflicts
```sh
# Open conflicting file and resolve issues manually
git add <conflicted_file>
git commit -m "Resolved merge conflict"
```

## 16. Undoing Commits
```sh
git reset --soft HEAD~1  # Undo last commit but keep changes
git reset --hard HEAD~1  # Undo last commit and remove changes
```

## 17. Setting Up Remote Repository
```sh
git remote add origin <repository_url>
git push -u origin main
```

## 18. Viewing and Changing Remote URLs
```sh
git remote -v  # View remote repository URLs
git remote set-url origin <new_url>  # Change remote URL
```

## 19. GitHub Authentication
If using HTTPS:
```sh
git config --global credential.helper store
```
If using SSH:
```sh
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"
cat ~/.ssh/id_rsa.pub  # Copy this key to GitHub SSH settings
```

## 20. Git Alias for Efficiency
Speed up workflow by creating aliases:
```sh
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.cm "commit -m"
```
Now you can use:
```sh
git st   # Instead of git status
git co   # Instead of git checkout
```

## 21. Advanced Log View
```sh
git log --graph --pretty=format:'%C(yellow)%h%Creset -%C(red)%d%Creset %s %C(blue)[%cn] %Cgreen(%cr)%Creset'
```
This displays commits in a graphical format.

## 22. Finding a Specific Commit
```sh
git log -S "specific_text"
```

## 23. Checking File History
```sh
git log -- <filename>
```

## 24. Removing Untracked Files
```sh
git clean -f -d  # Remove all untracked files and directories
```

---
### Need Help?
```sh
git help  # Show general help
git <command> --help  # Show help for a specific command
```
