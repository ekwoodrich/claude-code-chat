# VS Code Remote Session Support - Fix Documentation

## Problem Statement

The original claude-code-chat extension was not working properly in VS Code remote sessions (SSH, containers, WSL, etc.). The extension would look for the Claude Code binary on the local machine (where VS Code's UI runs) rather than on the remote machine (where the code actually lives).

### Root Cause
In VS Code remote sessions, extensions run in different contexts:
- **UI extensions**: Run locally on your client machine
- **Workspace extensions**: Run remotely on the server/container

The original extension was running as a UI extension, causing it to:
1. Check for `claude` command locally instead of remotely
2. Execute Claude Code on the wrong machine
3. Use incorrect path handling for remote environments

## Solution Approach

### 1. Force Extension to Run Remotely
**Change**: Added `extensionKind: ["workspace"]` to `package.json`

**Why**: This ensures the extension runs on the remote server where the code lives, not on the local VS Code client.

### 2. Remote-Aware Error Handling
**Change**: Enhanced Claude Code detection with remote session awareness

**Why**: Provides clearer error messages when Claude Code isn't found, specifically mentioning that it needs to be installed on the remote machine.

### 3. Path Handling for Remote Sessions
**Change**: Updated WSL path conversion logic to handle remote environments

**Why**: Remote sessions typically use Unix-style paths, so Windows-to-WSL conversion is unnecessary and potentially harmful.

### 4. Remote Session Detection
**Change**: Added `vscode.env.remoteName` detection throughout the codebase

**Why**: Allows the extension to adapt its behavior based on whether it's running in a remote session.

## Code Changes Made

### File: `package.json`
```json
// Added extension kind to force workspace execution
"extensionKind": [
  "workspace"
],

// Rebranded to avoid conflicts with store version
"name": "claude-code-chat-remote",
"displayName": "Claude Code Chat (Remote)",
"description": "Beautiful Claude Code Chat Interface for VS Code with Remote Session Support",
"publisher": "ekwoodrich",
"author": "Eliot Woodrich (fork of Andre Pimenta's extension)",
```

### File: `src/extension.ts`

#### Enhanced Error Handling (`line ~643`)
```typescript
// Check if claude command is not installed
if (error.message.includes('ENOENT') || error.message.includes('command not found')) {
  // Check if we're in a remote workspace
  const isRemote = vscode.env.remoteName !== undefined;
  const remoteMessage = isRemote ? 
    ' Make sure Claude Code is installed on the remote machine, not locally.' : '';
  
  this._sendAndSaveMessage({
    type: 'error',
    data: `Claude Code not found. Install claude code first: https://www.anthropic.com/claude-code${remoteMessage}`
  });
}
```

#### Remote-Aware Path Conversion (`line ~1793`)
```typescript
private convertToWSLPath(windowsPath: string): string {
  const config = vscode.workspace.getConfiguration('claudeCodeChat');
  const wslEnabled = config.get<boolean>('wsl.enabled', false);
  const isRemote = vscode.env.remoteName !== undefined;

  // In remote sessions, paths are likely already Unix-style, so WSL conversion may not be needed
  if (isRemote) {
    // For remote sessions, the path is likely already in the correct format for the remote environment
    return windowsPath;
  }

  if (wslEnabled && windowsPath.match(/^[a-zA-Z]:/)) {
    // Convert C:\Users\... to /mnt/c/Users/...
    return windowsPath.replace(/^([a-zA-Z]):/, '/mnt/$1').toLowerCase().replace(/\\/g, '/');
  }

  return windowsPath;
}
```

#### Platform Info with Remote Detection (`line ~2318`)
```typescript
private _sendPlatformInfo() {
  const platform = process.platform;
  const dismissed = this._context.globalState.get<boolean>('wslAlertDismissed', false);
  const isRemote = vscode.env.remoteName !== undefined;

  // Get WSL configuration
  const config = vscode.workspace.getConfiguration('claudeCodeChat');
  const wslEnabled = config.get<boolean>('wsl.enabled', false);

  this._postMessage({
    type: 'platformInfo',
    data: {
      platform: platform,
      isWindows: platform === 'win32',
      wslAlertDismissed: dismissed,
      wslEnabled: wslEnabled,
      isRemote: isRemote,
      remoteName: vscode.env.remoteName
    }
  });
}
```

## Installation Instructions

### For Remote VS Code Sessions:

1. **Clone the repository on your remote server**:
   ```bash
   git clone https://github.com/ekwoodrich/claude-code-chat.git
   cd claude-code-chat
   ```

2. **Install dependencies and compile**:
   ```bash
   npm install
   npm run compile
   ```

3. **Package the extension**:
   ```bash
   npx vsce package
   ```
   This creates a `.vsix` file.

4. **Install in VS Code**:
   - Open VS Code command palette (`Ctrl+Shift+P` / `Cmd+Shift+P`)
   - Run `Extensions: Install from VSIX...`
   - Select the generated `.vsix` file

5. **Ensure Claude Code is installed on the remote machine**:
   ```bash
   # Install Claude Code on the remote server, not locally
   npm install -g @anthropic-ai/claude-cli
   # or follow official installation instructions
   ```

## Key Benefits

1. **Side-by-side installation**: Can be installed alongside the original store version
2. **Remote-first design**: Properly detects and handles remote environments
3. **Better error messages**: Clear guidance about where Claude Code needs to be installed
4. **Path handling**: Correct path conversion for different remote environments
5. **Future-proof**: Easily extensible for other remote session types

## Testing Checklist

- [ ] Extension loads in remote VS Code session
- [ ] Claude Code detection works on remote machine
- [ ] Error messages mention remote installation requirements
- [ ] Path handling works correctly in remote environment
- [ ] Extension can be installed alongside store version
- [ ] MCP permissions system works in remote context

## Commits Made

1. **e9ad25f**: Add VS Code remote session support
   - Set extensionKind to "workspace"
   - Enhanced Claude Code detection with remote-aware error messages
   - Updated WSL path conversion for remote environments
   - Added remote session detection to platform info

2. **88e057f**: Rebrand extension for independent installation
   - Changed extension name to claude-code-chat-remote
   - Updated publisher and author information
   - Updated repository URL

## Future Considerations

1. **Container-specific optimizations**: Could add Docker/Podman container detection
2. **SSH key forwarding**: Potential integration with SSH agent forwarding
3. **Performance optimizations**: Remote-aware caching strategies
4. **Multi-workspace support**: Handle multiple remote workspaces

## Troubleshooting

### Extension not found after installation
- Verify the extension is installed with the name "Claude Code Chat (Remote)"
- Check that it shows publisher as "ekwoodrich"

### Claude Code still not found
- Ensure Claude Code is installed on the remote machine: `which claude`
- Check that the Claude binary is in the PATH of the remote environment
- Verify you're not trying to use a local Claude installation

### Path issues
- Check if WSL mode is incorrectly enabled for remote sessions
- Verify that file paths are being handled correctly in the remote environment

---

**Author**: Eliot Woodrich  
**Date**: 2025-08-07  
**Original Extension**: Andre Pimenta's claude-code-chat  
**Fork Repository**: https://github.com/ekwoodrich/claude-code-chat