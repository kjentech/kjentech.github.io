Why: Your organization might be using a web proxy with SSL inspection that's interfering with Git Credential Manager for Windows' authentication flow.

We assume that you have enabled the built-in OpenSSH Agent in Windows.

```
# Start service and set startuptype to Auto
Get-Service ssh-agent | Set-Service -StartupType Automatic -PassThru | Start-Service

ssh-keygen -t ed25519 -C "email@domain.com"

# ssh-add will pick up the key from %userprofile%\.ssh\ and prompt for password
ssh-add

# Set git to use the built-in OpenSSH agent to workaround a bug in VS Code
git config --global core.sshCommand "C:/Windows/System32/OpenSSH/ssh.exe"

Get-Service ssh-agent | Restart-Service
```