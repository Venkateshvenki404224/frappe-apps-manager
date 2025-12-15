# Frappe Apps Manager - Installation Guide

Complete guide for installing the Frappe Apps Manager plugin from GitHub.

---

## Quick Install (Recommended)

The easiest way to install from GitHub:

```bash
# 1. Start Claude Code
claude

# 2. Add marketplace using GitHub shorthand
/plugin marketplace add Venkateshvenki404224/frappe-apps-manager

# 3. Install the plugin
/plugin install frappe-apps-manager@Venkateshvenki404224

# 4. Restart Claude Code to load the plugin

# 5. Verify installation
/help  # Should show all 15 Frappe commands
```

**That's it!** The plugin is now installed and ready to use.

---

## Installation Methods

### Method 1: GitHub Organization/Repo (Recommended)

**Format:** `owner/repository`

```bash
/plugin marketplace add Venkateshvenki404224/frappe-apps-manager
/plugin install frappe-apps-manager@Venkateshvenki404224
```

**Advantages:**
- ✅ Shortest command
- ✅ Automatic GitHub resolution
- ✅ Always gets latest version
- ✅ Easiest to remember

**When to use:**
- First-time installation
- Regular users
- Team setup

---

### Method 2: Full GitHub URL

**Format:** `https://github.com/owner/repository`

```bash
/plugin marketplace add https://github.com/Venkateshvenki404224/frappe-apps-manager
/plugin install frappe-apps-manager@Venkateshvenki404224
```

**Advantages:**
- ✅ Explicit GitHub URL
- ✅ Works with any Git hosting
- ✅ Clear what's being installed

**When to use:**
- Want to be explicit
- Using custom Git hosting
- Copying from documentation

---

### Method 3: SSH URL (For Contributors)

**Format:** `git@github.com:owner/repository.git`

```bash
/plugin marketplace add git@github.com:Venkateshvenki404224/frappe-apps-manager.git
/plugin install frappe-apps-manager@Venkateshvenki404224
```

**Advantages:**
- ✅ Uses SSH authentication
- ✅ Better for private repositories
- ✅ Required for push access

**When to use:**
- Contributing to the plugin
- Have SSH keys configured
- Working with private forks

---

### Method 4: Local Development

**Format:** `./path/to/marketplace`

```bash
# Clone repository first
git clone https://github.com/Venkateshvenki404224/frappe-apps-manager.git
cd frappe-apps-manager

# Then in Claude Code
/plugin marketplace add ./frappe-apps-manager
/plugin install frappe-apps-manager@frappe-apps-manager
```

**Advantages:**
- ✅ Test local changes
- ✅ Development workflow
- ✅ No network dependency

**When to use:**
- Plugin development
- Testing local changes
- Offline development

---

## Step-by-Step Installation

### For First-Time Users

**Step 1: Start Claude Code**
```bash
# In your terminal
claude
```

**Step 2: Add the Marketplace**

In Claude Code, run:
```
/plugin marketplace add Venkateshvenki404224/frappe-apps-manager
```

You'll see:
```
✓ Marketplace added successfully
```

**Step 3: Install the Plugin**
```
/plugin install frappe-apps-manager@Venkateshvenki404224
```

Select "Install now" when prompted.

**Step 4: Restart Claude Code**
```
exit
claude
```

Or press `Ctrl+C` and restart.

**Step 5: Verify Installation**
```
/help
```

You should see all Frappe commands:
```
Frappe Commands:
  /frappe-new-app          Create a new Frappe application
  /frappe-new-site         Create a new Frappe site
  /frappe-install-app      Install Frappe app to site
  /frappe-bench-start      Start Frappe bench
  /frappe-migrate          Run database migrations
  /frappe-backup           Backup Frappe site
  /frappe-new-doctype      Create new DocType
  /frappe-deploy           Deploy to production
  /frappe-test             Run tests with coverage
  /frappe-logs             View and analyze logs
  /frappe-console          Interactive Python console
  /frappe-cache            Manage Redis cache
  /frappe-role-manager     Manage roles and permissions
  /frappe-config           Configure site settings
  /frappe-db-reset         Reset database for development
```

**Step 6: Try It Out**
```
/frappe-new-site
```

Follow the prompts to create your first Frappe site!

---

## Team Installation

### Option A: Repository-Level Auto-Install (Recommended for Teams)

**Setup once in your project:**

1. Create `.claude/settings.json` in your project root:
```json
{
  "pluginMarketplaces": [
    {
      "name": "frappe-marketplace",
      "source": "Venkateshvenki404224/frappe-apps-manager"
    }
  ],
  "plugins": [
    {
      "name": "frappe-apps-manager",
      "marketplace": "frappe-marketplace",
      "enabled": true
    }
  ]
}
```

2. Commit to your repository:
```bash
git add .claude/settings.json
git commit -m "chore: add Frappe Apps Manager plugin"
git push
```

3. **Team members automatically get the plugin:**
   - They clone the repository
   - Trust the repository folder in Claude Code settings
   - Plugin installs automatically when they start Claude Code

**Benefits:**
- ✅ Consistent tooling across team
- ✅ No manual installation needed
- ✅ Version controlled
- ✅ Same setup for everyone

---

### Option B: User-Level Install (Personal Use)

Install for all your projects:

```bash
# Add to user settings
/plugin marketplace add Venkateshvenki404224/frappe-apps-manager --scope user
/plugin install frappe-apps-manager@Venkateshvenki404224 --scope user
```

**Benefits:**
- ✅ Available in all projects
- ✅ Personal productivity tool
- ✅ Install once, use everywhere

---

## Verify Installation

### Check Marketplace
```
/plugin marketplace list
```

Should show:
```
frappe-marketplace (Venkateshvenki404224/frappe-apps-manager)
```

### Check Installed Plugins
```
/plugin
```

Select "Manage Plugins" to see:
```
frappe-apps-manager (v2.0.0) - Enabled
  Source: Venkateshvenki404224/frappe-apps-manager
  Commands: 15
  Agents: 10
  Skills: 15
```

### Check Commands
```
/help
```

Look for "Frappe Commands" section with all 15 commands.

### Check Agents
```
/agents
```

Should show 10 Frappe agents.

### Check Output Styles
```
/output-style
```

Should show "Frappe Conventional Commits" option.

---

## Troubleshooting

### Plugin Not Installing

**Problem:** "Marketplace not found"
**Solution:**
```bash
# Verify marketplace name
/plugin marketplace list

# Try full URL
/plugin marketplace add https://github.com/Venkateshvenki404224/frappe-apps-manager
```

**Problem:** "Network error"
**Solution:**
```bash
# Check internet connection
ping github.com

# Try again
/plugin marketplace add Venkateshvenki404224/frappe-apps-manager
```

**Problem:** "Permission denied"
**Solution:**
```bash
# For private repos, configure SSH
ssh -T git@github.com

# Use SSH URL
/plugin marketplace add git@github.com:Venkateshvenki404224/frappe-apps-manager.git
```

### Commands Not Showing

**Problem:** Commands not in `/help`
**Solution:**
```bash
# Restart Claude Code
exit
claude

# Verify plugin enabled
/plugin
# Check that frappe-apps-manager shows as "Enabled"
```

**Problem:** "Command not found"
**Solution:**
```bash
# Reinstall plugin
/plugin uninstall frappe-apps-manager@Venkateshvenki404224
/plugin install frappe-apps-manager@Venkateshvenki404224

# Restart Claude Code
```

### Skills Not Working

**Problem:** Skills not invoking
**Solution:**
- Skills invoke automatically based on context
- Just describe what you want naturally
- Example: "Create a client script for my Customer form"
- Claude will automatically use frappe-client-script-generator skill

### Update Plugin

**Get latest version:**
```bash
# Uninstall current
/plugin uninstall frappe-apps-manager@Venkateshvenki404224

# Reinstall (gets latest)
/plugin install frappe-apps-manager@Venkateshvenki404224

# Restart Claude Code
```

---

## Installation Scopes

### Project Scope (Default)
```bash
/plugin marketplace add Venkateshvenki404224/frappe-apps-manager
# Available only in current project
```

### User Scope (All Projects)
```bash
/plugin marketplace add Venkateshvenki404224/frappe-apps-manager --scope user
# Available in all your projects
```

### Local Scope (Project-Specific)
```bash
/plugin marketplace add Venkateshvenki404224/frappe-apps-manager --scope local
# Only for you in this project
```

---

## Uninstallation

### Remove Plugin
```bash
/plugin uninstall frappe-apps-manager@Venkateshvenki404224
```

### Remove Marketplace
```bash
/plugin marketplace remove Venkateshvenki404224
```

### Complete Removal
```bash
# Uninstall plugin
/plugin uninstall frappe-apps-manager@Venkateshvenki404224

# Remove marketplace
/plugin marketplace remove Venkateshvenki404224

# Restart Claude Code
exit
claude
```

---

## Quick Reference

### Installation Commands

```bash
# Quickest method
/plugin marketplace add Venkateshvenki404224/frappe-apps-manager
/plugin install frappe-apps-manager@Venkateshvenki404224

# Full URL method
/plugin marketplace add https://github.com/Venkateshvenki404224/frappe-apps-manager
/plugin install frappe-apps-manager@Venkateshvenki404224

# SSH method (for contributors)
/plugin marketplace add git@github.com:Venkateshvenki404224/frappe-apps-manager.git
/plugin install frappe-apps-manager@Venkateshvenki404224

# Local method (for development)
git clone https://github.com/Venkateshvenki404224/frappe-apps-manager.git
/plugin marketplace add ./frappe-apps-manager
/plugin install frappe-apps-manager@frappe-apps-manager
```

### Management Commands

```bash
# List marketplaces
/plugin marketplace list

# List installed plugins
/plugin

# Enable/disable
/plugin enable frappe-apps-manager@Venkateshvenki404224
/plugin disable frappe-apps-manager@Venkateshvenki404224

# Update plugin
/plugin uninstall frappe-apps-manager@Venkateshvenki404224
/plugin install frappe-apps-manager@Venkateshvenki404224
```

---

## Support

**Issues:** https://github.com/Venkateshvenki404224/frappe-apps-manager/issues
**Discussions:** https://github.com/Venkateshvenki404224/frappe-apps-manager/discussions
**Documentation:** See [README.md](README.md) and [CLAUDE.md](CLAUDE.md)

---

## What You Get

After installation, you get:

✅ **15 Commands** - Complete Frappe workflow
✅ **10 Agents** - Expert assistance
✅ **15 Skills** - Autonomous code generation
✅ **1 Output Style** - Frappe commit standard
✅ **7 Hooks** - Smart automation

All with **real patterns from ERPNext** and **95% SDLC coverage**!

---

**Repository:** https://github.com/Venkateshvenki404224/frappe-apps-manager
**Version:** 2.0.0
**License:** MIT
**Status:** Production Ready 🚀
