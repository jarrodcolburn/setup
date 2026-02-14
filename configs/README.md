# Configuration Management

This directory contains configuration templates for various applications. These files are intended to be deployed to the user's home directory.

## Directory Structure
- `configs/<app>/`: Configuration files for a specific application.

## How to Use as Templates
1. **Modify**: Edit the files in this directory to set your preferred defaults.
2. **Deploy**: Copy or symlink the files to your home directory.
   - Example: `cp configs/zsh/p10k.zsh ~/.p10k.zsh`
3. **Automate**: Add deployment logic to `setup.md` to ensure configurations are applied during system provisioning.

## Best Practices
- **Variables**: Use placeholders (e.g., `{{USERNAME}}`) if you plan to use a templating engine (like `jinja2` or `envsubst`) in the future.
- **Backups**: Always back up existing configurations before overwriting them.
- **Selective Sync**: Only track settings that are non-sensitive and common across environments.
