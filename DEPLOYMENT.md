# ProxmoxMCP-Plus Deployment Guide

## Issue Fixed: Missing fastmcp Dependency

**Date:** 2026-05-26  
**Fixed by:** Ultron  
**Branch:** `fix-fastmcp-dependency`  
**Commit:** 48b59ff

### Problem
The ProxmoxMCP-Plus container was crashing with:
```
ModuleNotFoundError: No module named 'mcp.server.fastmcp'
```

### Root Cause
The application code imports `from mcp.server.fastmcp import FastMCP` but the `fastmcp` package was not listed in the project dependencies.

- `fastmcp` is a separate PyPI package (version 3.3.1), not part of the core MCP SDK
- The dependency was missing from both `pyproject.toml` and `requirements.in`

### Solution
Added `fastmcp>=3.0.0` to the project dependencies:

**Files modified:**
- `pyproject.toml` - Added to `dependencies` list
- `requirements.in` - Added for pip-tools compatibility

**Verification:**
```bash
# Confirmed fastmcp is now installed in the rebuilt image:
docker run --rm registry.loc.wallacearizona.us/proxmox-mcp-plus:latest \
  /bin/bash -c '. .venv/bin/activate && pip list | grep fast'

# Output:
fastapi                   0.136.3
fastmcp                   3.3.1    ✅
fastmcp-slim              3.3.1    ✅
```

---

## Deployment Instructions

### Option 1: Build and Push from Local Machine (RECOMMENDED)

```bash
# 1. Clone the repository with the fix
git clone https://github.com/brydenver2/ProxmoxMCP-Plus.git
cd ProxmoxMCP-Plus
git checkout fix-fastmcp-dependency

# 2. Build the Docker image
docker build -t registry.loc.wallacearizona.us/proxmox-mcp-plus:latest .

# 3. Login to the private registry
docker login registry.loc.wallacearizona.us
# Username: (your registry username)
# Password: (your registry password)

# 4. Push to registry
docker push registry.loc.wallacearizona.us/proxmox-mcp-plus:latest

# 5. Update the swarm service
ssh ultron@wolverine.loc.wallacearizona.us \
  "docker service update --image registry.loc.wallacearizona.us/proxmox-mcp-plus:latest --force mcp-servers_proxmox-mcp-plus"

# 6. Verify deployment
ssh ultron@wolverine.loc.wallacearizona.us \
  "docker service ps mcp-servers_proxmox-mcp-plus && docker service logs --tail 20 mcp-servers_proxmox-mcp-plus"
```

### Option 2: Build on Wolverine (if registry push not available)

```bash
# 1. SSH to wolverine
ssh ultron@wolverine.loc.wallacearizona.us

# 2. Clone and build
cd /tmp
git clone https://github.com/brydenver2/ProxmoxMCP-Plus.git
cd ProxmoxMCP-Plus
git checkout fix-fastmcp-dependency
docker build -t registry.loc.wallacearizona.us/proxmox-mcp-plus:latest .

# 3. Save the image
docker save registry.loc.wallacearizona.us/proxmox-mcp-plus:latest -o /tmp/proxmox-mcp-plus-fixed.tar

# 4. Copy to thor (the node where service runs)
scp /tmp/proxmox-mcp-plus-fixed.tar ultron@thor.loc.wallacearizona.us:/tmp/

# 5. Load on thor
ssh ultron@thor.loc.wallacearizona.us "docker load -i /tmp/proxmox-mcp-plus-fixed.tar"

# 6. Force service update
docker service update --image registry.loc.wallacearizona.us/proxmox-mcp-plus:latest --force mcp-servers_proxmox-mcp-plus

# 7. Verify
docker service ps mcp-servers_proxmox-mcp-plus
docker service logs --tail 50 mcp-servers_proxmox-mcp-plus
```

### Option 3: Merge and CI/CD (BEST for production)

```bash
# 1. Create a pull request on GitHub
# Visit: https://github.com/brydenver2/ProxmoxMCP-Plus/pull/new/fix-fastmcp-dependency

# 2. Review and merge the PR

# 3. If you have GitHub Actions configured:
# - The CI/CD pipeline will build and push the image automatically
# - Monitor the Actions tab for build status

# 4. Once built, update the service:
ssh ultron@wolverine.loc.wallacearizona.us \
  "docker service update --image registry.loc.wallacearizona.us/proxmox-mcp-plus:latest --force mcp-servers_proxmox-mcp-plus"
```

---

## Verification Steps

After deployment, verify the service is healthy:

```bash
# 1. Check service status
docker service ps mcp-servers_proxmox-mcp-plus

# Expected: 1/1 replicas running on thor

# 2. Check logs for successful startup
docker service logs --tail 50 mcp-servers_proxmox-mcp-plus

# Look for:
# - "Starting MCP OpenAPI Proxy on 0.0.0.0:8811"
# - "Uvicorn server starting..."
# - "Application startup complete"
# - NO "ModuleNotFoundError" messages

# 3. Test the health endpoint
curl -k https://wamcp-proxmox-plus.local.wallacearizona.us/health

# Expected: {"status": "healthy"}

# 4. Check available tools
curl -k https://wamcp-proxmox-plus.local.wallacearizona.us/tools

# Expected: JSON array of Proxmox MCP tools
```

---

## Rollback Plan

If the new image causes issues:

```bash
# 1. Get the previous image digest
docker service inspect mcp-servers_proxmox-mcp-plus --format '{{.PreviousSpec.TaskTemplate.ContainerSpec.Image}}'

# 2. Rollback to previous version
docker service update --rollback mcp-servers_proxmox-mcp-plus

# 3. Verify rollback
docker service ps mcp-servers_proxmox-mcp-plus
```

---

## Future Prevention

To prevent similar dependency issues:

1. **Add dependency verification to CI/CD:**
   ```bash
   # In GitHub Actions or pre-commit hooks:
   uv pip compile --dry-run
   uv pip check
   ```

2. **Test imports during build:**
   ```dockerfile
   # Add to Dockerfile:
   RUN . .venv/bin/activate && python -c "from mcp.server.fastmcp import FastMCP"
   ```

3. **Keep dependencies synchronized:**
   ```bash
   # Regenerate requirements.txt from pyproject.toml:
   uv pip compile pyproject.toml -o requirements.txt
   ```

---

## Support

**Questions or issues?**
- Contact: Ultron (ultron@aquabutlers.com)
- GitHub Issue: https://github.com/brydenver2/ProxmoxMCP-Plus/issues
- Swarm Manager: ultron@wolverine.loc.wallacearizona.us

**Service Details:**
- Service name: `mcp-servers_proxmox-mcp-plus`
- Stack: `mcp-servers`
- Node: thor (manager node)
- Endpoint: https://wamcp-proxmox-plus.local.wallacearizona.us
- Internal port: 8811
