🔐 DevSecOps for Git

📘 Introduction:

DevSecOps for Git means integrating security practices directly into the Git workflow to ensure code remains secure from development through deployment.

🔄 DevSecOps Workflow:
Write Code → Commit → Push → Scan → Review → Merge → Deploy
Security is enforced at every stage of this pipeline.

📁 .gitignore:
📌 Purpose

Prevents sensitive or unnecessary files from being tracked by Git.

📄 Example:
.env
node_modules/
*.log
✅ Benefits
Prevents leaking sensitive data
Keeps repository clean
🪝 Git Hooks
📌 Definition

Hooks are scripts that run automatically before or after Git actions.

🔍 Types
Pre-commit → Runs before commit
Pre-push → Runs before push
✅ Use Cases
Code quality checks
Prevent committing secrets
⚙️ Pre-Commit Framework
📌 Definition

A framework to manage Git hooks efficiently.

🔧 Features
Easy setup
Reusable hooks
Automated checks
📄 Example Configuration
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.0.0
    hooks:
      - id: gitleaks
🔐 Gitleaks
📌 Definition

Gitleaks is a tool used to detect sensitive data in code.

🔍 Detects
API keys
Passwords
Tokens
🛠️ Usage
Scan Repository
gitleaks detect
Generate Report
gitleaks detect -r report.json
Scan Git History
gitleaks detect --log-opts="--all"
🛡️ Branch Protection Rules
📌 Purpose

Protect important branches like main.

🔧 Rules
No direct push
Require pull requests
Require status checks
👥 RBAC (Role-Based Access Control)
📌 Definition

Access is granted based on user roles.

🎯 Example
Developer → Push code
Admin → Merge code
👀 Mandatory Reviews
📌 Purpose

Ensure code is reviewed before merging.

✅ Benefits
Detect bugs
Improve security
📄 CODEOWNERS
📌 Definition

Defines who is responsible for specific parts of the codebase.

📄 Example
# Assign owners to specific paths
*       @team-leads
/src/   @dev-team
🚀 Summary

DevSecOps in Git = Prevent + Detect + Protect + Automate
