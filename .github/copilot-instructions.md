# Verdaccio Development Instructions

**Always follow these instructions first and fallback to search or bash commands only when you encounter unexpected information that does not match the info here.**

Verdaccio is a lightweight private npm registry built with Node.js, TypeScript, React, and pnpm. This repository is a monorepo containing multiple packages including the core server, plugins, CLI tools, and a React-based web UI.

## Working Effectively

### Prerequisites and Environment Setup
- Install Node.js version 14 (check `.nvmrc` for exact version): `nvm install`
- Install pnpm globally: `npm i -g pnpm@latest`

### Bootstrap and Build Process
Follow these steps in exact order:

1. **Fix Registry Configuration** (Required for external development):
   ```bash
   sed -i 's|registry = https://registry.verdaccio.org|registry = https://registry.npmjs.org|' .npmrc
   ```

2. **Fix Workspace Dependencies** (Required for external development):
   ```bash
   # Update verdaccio package
   sed -i 's|"verdaccio-audit": "11.0.0-alpha.4"|"verdaccio-audit": "workspace:*"|' packages/verdaccio/package.json
   sed -i 's|"verdaccio-htpasswd": "11.0.0-alpha.6"|"verdaccio-htpasswd": "workspace:*"|' packages/verdaccio/package.json
   # Update server package  
   sed -i 's|"verdaccio-audit": "workspace:11.0.0-alpha.4"|"verdaccio-audit": "workspace:*"|' packages/server/package.json
   ```

3. **Install Dependencies**:
   ```bash
   rm -f pnpm-lock.yaml
   pnpm install --ignore-scripts
   ```
   - **TIMING**: Takes 2-3 minutes. NEVER CANCEL. Set timeout to 10+ minutes.
   - **NEVER CANCEL**: Build may fail on first attempt due to network issues with private registry.

4. **Build JavaScript (Core packages)**:
   ```bash
   pnpm recursive run build:js
   ```
   - **TIMING**: Takes 10-15 seconds. NEVER CANCEL. Set timeout to 5+ minutes.
   - **IMPORTANT**: TypeScript compilation (`pnpm build`) fails due to type compatibility issues. Always use `build:js` only.

5. **Build UI Theme** (Required for web interface):
   ```bash
   cd packages/plugins/ui-theme
   NODE_OPTIONS="--openssl-legacy-provider" pnpm build
   cd ../../..
   ```
   - **TIMING**: Takes 12-15 seconds. NEVER CANCEL. Set timeout to 10+ minutes.
   - **CRITICAL**: Requires `NODE_OPTIONS="--openssl-legacy-provider"` for Node.js v20 compatibility.

### Running the Application

**Start Verdaccio Server**:
```bash
NODE_OPTIONS="--openssl-legacy-provider" node packages/verdaccio/bin/verdaccio --listen 4873
```
- Server starts on `http://localhost:4873`
- **TIMING**: Server starts in 3-5 seconds
- **VALIDATION**: Test with `curl -I http://localhost:4873` - should return HTTP 200
- **UI ACCESS**: Open `http://localhost:4873` in browser to access React-based UI

### Testing and Validation

**Linting**:
```bash
pnpm lint
```
- **TIMING**: Takes 15-20 seconds. Set timeout to 5+ minutes.
- **VALIDATION**: Should pass with warnings (49 warnings are expected)

**Formatting Check**:
```bash
pnpm format:check
```
- **TIMING**: Takes 8-10 seconds. Set timeout to 3+ minutes.

**Apply Formatting**:
```bash
pnpm format
```

### Known Limitations and Issues

**DO NOT attempt the following** (these commands fail in this environment):

- `pnpm build` - TypeScript compilation fails due to type compatibility
- `pnpm test` - Babel version compatibility issues prevent testing
- `pnpm test:e2e:cli` - Babel compatibility issues
- `pnpm test:e2e:ui` - Babel compatibility issues  
- `pnpm start:ts` - TypeScript runtime errors
- `pnpm start` - Depends on successful build

**Registry Issues**:
- Project is configured for private Verdaccio registry (`https://registry.verdaccio.org`) which is not accessible externally
- **ALWAYS** change to `https://registry.npmjs.org` for external development
- This is documented as a known workaround, not a bug

## Validation Scenarios

**ALWAYS perform these validation steps after making changes**:

1. **Build Validation**:
   ```bash
   pnpm recursive run build:js
   NODE_OPTIONS="--openssl-legacy-provider" pnpm build --filter ...@verdaccio/ui-theme
   ```

2. **Server Functionality Test**:
   ```bash
   # Start server in background
   NODE_OPTIONS="--openssl-legacy-provider" node packages/verdaccio/bin/verdaccio --listen 4873 &
   SERVER_PID=$!
   sleep 10
   # Test API endpoints
   curl -I http://localhost:4873
   curl -s http://localhost:4873 | head -10
   # Stop server
   kill $SERVER_PID
   ```

3. **Code Quality Validation**:
   ```bash
   pnpm lint
   pnpm format:check
   ```

## Project Structure

**Key Directories**:
- `packages/` - Monorepo packages (core, plugins, tools)
- `packages/verdaccio/` - Main CLI application 
- `packages/server/` - HTTP server implementation
- `packages/plugins/ui-theme/` - React-based web UI
- `packages/api/` - REST API implementation
- `test/e2e-cli/` - End-to-end CLI tests (not functional)
- `test/e2e-ui/` - End-to-end UI tests (not functional)

**Important Files**:
- `.nvmrc` - Required Node.js version (14)
- `.npmrc` - Registry configuration (requires modification)
- `pnpm-workspace.yaml` - Workspace package definitions
- `package.json` - Root project configuration

## Common Development Tasks

**Adding New Dependencies**:
```bash
# Add to specific package
cd packages/[package-name]
pnpm add [dependency]
# Add to root
pnpm add -w [dependency]
```

**Working with Specific Packages**:
```bash
cd packages/[package-name]
pnpm build:js
pnpm lint
```

**Debugging**:
- Use `DEBUG=verdaccio:* node packages/verdaccio/bin/verdaccio` for verbose logging
- Server logs appear in console when running
- Check browser developer tools for UI issues

## CI/Build Pipeline Integration

**Pre-commit Validation** (Always run before committing):
```bash
pnpm lint
pnpm format
pnpm recursive run build:js
NODE_OPTIONS="--openssl-legacy-provider" pnpm build --filter ...@verdaccio/ui-theme
```

**Expected Build Times**:
- Installation: 2-3 minutes
- JavaScript Build: 10-15 seconds  
- UI Build: 12-15 seconds
- Linting: 15-20 seconds
- Formatting: 8-10 seconds

**CRITICAL TIMEOUTS**:
- Set all build timeouts to 10+ minutes minimum
- Set installation timeouts to 15+ minutes minimum
- NEVER CANCEL long-running operations

This development environment has specific compatibility constraints. Follow these exact instructions for reliable builds and execution.