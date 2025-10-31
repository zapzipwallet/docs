# Fixing Chrome Extension Webpack Bundling Issues: A Deep Dive into Polyfills and Module Resolution

## The Problem

When building a Chrome extension that uses Node.js cryptographic libraries (like `@solana/web3.js`, `bip39`, and `ed25519-hd-key`), you'll encounter several critical errors that can be confusing to debug:

1. **`Uncaught ReferenceError: process is not defined`**
2. **`TypeError: Cannot read properties of undefined (reading 'call')`** when using ed25519-hd-key
3. **Module resolution failures** for Node.js built-in modules like `crypto`, `stream`, `buffer`

This guide documents the complete solution for the ZAP Wallet Chrome Extension.

---

## Understanding the Root Causes

### 1. Browser vs Node.js Environment

Chrome extensions run in a browser environment, not Node.js. This means:
- **No `process` global**: Node.js has a global `process` object that many libraries check
- **No built-in crypto module**: Node's `crypto` module doesn't exist in browsers
- **No Buffer constructor**: While available in Node, `Buffer` needs to be polyfilled for browsers
- **No stream module**: File streaming works differently in browsers

### 2. Webpack Module Resolution

Webpack bundles modules for the browser, but:
- It doesn't automatically polyfill Node.js globals
- CommonJS modules (`require`/`exports`) need special handling
- Some modules use dynamic imports that webpack can't resolve at build time
- The `fallback` configuration tells webpack what to use instead of Node modules

### 3. The ed25519-hd-key Problem

The `ed25519-hd-key` library is particularly problematic because:
- It's a CommonJS module (`exports.derivePath = ...`)
- It depends on `create-hmac` which requires the `crypto` module
- It depends on `tweetnacl` for cryptographic operations
- Webpack's default configuration sets `crypto: false`, completely excluding it

---

## The Solution: Step-by-Step

### Step 1: Install Required Polyfills

```bash
npm install --save-dev process crypto-browserify stream-browserify
npm install create-hmac tweetnacl
```

**Why each package:**
- `process`: Provides the `process` global for browser environments
- `crypto-browserify`: Pure JavaScript implementation of Node's crypto module
- `stream-browserify`: Stream implementation for browsers
- `create-hmac`: HMAC implementation (dependency of ed25519-hd-key)
- `tweetnacl`: NaCl cryptography library (dependency of ed25519-hd-key)

### Step 2: Configure Webpack Properly

```javascript
// webpack.config.js
const path = require('path');
const webpack = require('webpack');

module.exports = {
  entry: {
    popup: './src/popup-bundle.js',
    background: './background/background.js'
  },
  output: {
    path: path.resolve(__dirname, 'popup'),
    filename: '[name]-bundle.js',
    clean: false
  },
  resolve: {
    fallback: {
      // CRITICAL: Enable crypto and stream polyfills
      "crypto": require.resolve("crypto-browserify"),
      "stream": require.resolve("stream-browserify"),
      
      // These can stay false if not needed
      "assert": false,
      "http": false,
      "https": false,
      "os": false,
      "url": false,
      "zlib": false,
      "path": false,
      "fs": false,
      
      // Required polyfills
      "buffer": require.resolve("buffer/"),
      "process": require.resolve("process/browser")
    }
  },
  plugins: [
    // CRITICAL: Provide global variables that libraries expect
    new webpack.ProvidePlugin({
      process: 'process/browser',
      Buffer: ['buffer', 'Buffer']
    })
  ],
  mode: 'production',
  optimization: {
    minimize: true
  }
};
```

**Key Points:**
- `ProvidePlugin` injects globals **automatically** - you don't need to import them
- `fallback` tells webpack **what to use** when code imports Node modules
- Setting to `false` **completely excludes** that module (dangerous if dependencies need it)

### Step 3: Handle ed25519-hd-key Module Resolution

The `ed25519-hd-key` library presented a unique challenge. Even with proper webpack configuration, it wasn't being bundled correctly due to webpack's module resolution with CommonJS exports.

**Three approaches we tried:**

#### Approach 1: ES6 Import (Failed)
```javascript
import { derivePath } from 'ed25519-hd-key';
```
**Result:** Webpack couldn't resolve the module properly in production mode.

#### Approach 2: CommonJS require (Failed)
```javascript
const { derivePath } = require('ed25519-hd-key');
```
**Result:** Mixed ES6/CommonJS caused bundling issues, `derivePath` was undefined at runtime.

#### Approach 3: Inline Implementation (SUCCESS ✅)

We inlined the entire implementation:

```javascript
import createHmac from 'create-hmac';
import nacl from 'tweetnacl';

const ED25519_CURVE = 'ed25519 seed';
const HARDENED_OFFSET = 0x80000000;

const pathRegex = new RegExp("^m(\\/[0-9]+')+$");
const replaceDerive = (val) => val.replace("'", '');

const getMasterKeyFromSeed = (seed) => {
  const hmac = createHmac('sha512', ED25519_CURVE);
  const I = hmac.update(Buffer.from(seed, 'hex')).digest();
  const IL = I.slice(0, 32);
  const IR = I.slice(32);
  return { key: IL, chainCode: IR };
};

const CKDPriv = ({ key, chainCode }, index) => {
  const indexBuffer = Buffer.allocUnsafe(4);
  indexBuffer.writeUInt32BE(index, 0);
  const data = Buffer.concat([Buffer.alloc(1, 0), key, indexBuffer]);
  const I = createHmac('sha512', chainCode).update(data).digest();
  const IL = I.slice(0, 32);
  const IR = I.slice(32);
  return { key: IL, chainCode: IR };
};

const isValidPath = (path) => {
  if (!pathRegex.test(path)) return false;
  return !path.split('/').slice(1).map(replaceDerive).some(isNaN);
};

const derivePath = (path, seed, offset = HARDENED_OFFSET) => {
  if (!isValidPath(path)) {
    throw new Error('Invalid derivation path');
  }
  const { key, chainCode } = getMasterKeyFromSeed(seed);
  const segments = path.split('/').slice(1)
    .map(replaceDerive)
    .map(el => parseInt(el, 10));
  return segments.reduce(
    (parentKeys, segment) => CKDPriv(parentKeys, segment + offset),
    { key, chainCode }
  );
};
```

**Why this works:**
- Direct ES6 imports of `create-hmac` and `tweetnacl` work with webpack
- No CommonJS module resolution issues
- Crypto and stream polyfills are properly bundled
- Total control over the implementation

---

## Verification

### Bundle Size Changes
- **Before polyfills:** 553 KiB
- **After polyfills:** 588 KiB (+35 KiB for crypto modules)

### What's in the Bundle
You can verify the fix by checking the bundled code:

```bash
# Check for inlined ed25519 code
grep "ed25519 seed" popup/popup-bundle.js

# Check for create-hmac usage
grep "sha512" popup/popup-bundle.js

# Verify derivePath function exists
grep "Invalid derivation path" popup/popup-bundle.js
```

---

## Common Pitfalls to Avoid

### ❌ DON'T: Use DefinePlugin for process
```javascript
// This only defines process.env, not the process global
new webpack.DefinePlugin({
  'process.env': JSON.stringify({})
})
```

### ✅ DO: Use ProvidePlugin
```javascript
// This provides the entire process object
new webpack.ProvidePlugin({
  process: 'process/browser'
})
```

### ❌ DON'T: Set crypto to false if you need it
```javascript
fallback: {
  "crypto": false  // This breaks create-hmac!
}
```

### ✅ DO: Use crypto-browserify
```javascript
fallback: {
  "crypto": require.resolve("crypto-browserify")
}
```

### ❌ DON'T: Mix ES6 and CommonJS for problematic modules
```javascript
// Causes undefined at runtime
import x from 'module';
const { y } = require('other-module');
```

### ✅ DO: Use consistent import style or inline
```javascript
// Either all ES6
import x from 'module';
import { y } from 'other-module';

// Or inline the problematic module
const inlinedFunction = () => { /* implementation */ };
```

---

## Testing Your Fix

1. **Build the bundle:**
   ```bash
   npm run build
   ```

2. **Check for errors in webpack output:**
   - No "module not found" errors
   - Bundle size should increase (means modules are included)

3. **Load in Chrome:**
   - Go to `chrome://extensions/`
   - Enable Developer Mode
   - Load unpacked extension
   - Check Console (F12) for errors

4. **Test functionality:**
   - Generate a wallet
   - Check console logs for successful generation
   - Verify the public/private keys are created

---

## Bundle Size Optimization (Optional)

If your bundle gets too large (>1MB), consider:

1. **Code splitting:**
   ```javascript
   optimization: {
     splitChunks: {
       chunks: 'all',
     }
   }
   ```

2. **Lazy loading:**
   ```javascript
   // Instead of:
   import heavyModule from 'heavy-module';
   
   // Use:
   const heavyModule = await import('heavy-module');
   ```

3. **Tree shaking:**
   - Ensure `mode: 'production'`
   - Use named imports: `import { specific } from 'module'`

---

## Additional Notes

### About Test Key Warnings

Chrome will warn about test `.pem` files in `node_modules/public-encrypt/test/`. These are:
- Only test fixtures from the `public-encrypt` package
- Not actually used by your extension
- Safe to ignore (they're in node_modules, not your source)

To suppress these warnings, you can either:
1. **Ignore them** (recommended - they don't affect functionality)
2. **Exclude node_modules from packaging** when distributing your extension
3. **Use a custom build script** to copy only necessary files

---

## Final Architecture

```
Your Extension Code
       ↓
  popup-bundle.js  (ES6 imports)
       ↓
  Webpack Bundler
       ↓
  ├─ crypto-browserify (polyfill)
  ├─ stream-browserify (polyfill)
  ├─ process/browser (polyfill)
  ├─ buffer (polyfill)
  ├─ create-hmac (works with crypto-browserify)
  ├─ tweetnacl (pure JS crypto)
  └─ @solana/web3.js (Solana library)
       ↓
  popup-bundle.js (588 KiB, fully bundled)
       ↓
  Chrome Extension Environment ✅
```

---

## Key Takeaways

1. **Always enable required polyfills** - Don't set them to `false` unless you're certain they're not needed
2. **Use ProvidePlugin for globals** - DefinePlugin only works for compile-time replacements
3. **When imports fail, inline the code** - Sometimes it's simpler than fighting module resolution
4. **Check bundle size changes** - An increase means modules are being included (good!)
5. **Test in the actual environment** - Webpack builds might succeed but fail at runtime

---

## Troubleshooting Guide

| Error | Cause | Solution |
|-------|-------|----------|
| `process is not defined` | No process global | Add ProvidePlugin with process/browser |
| `crypto module not found` | crypto set to false | Use crypto-browserify |
| `Cannot read properties of undefined` | Module not bundled | Check fallback config, consider inlining |
| Bundle size doesn't change | Modules not included | Check fallback settings, verify imports |
| `Buffer is not defined` | No Buffer global | Add Buffer to ProvidePlugin |

---

## Resources

- [Webpack Resolve Fallback Documentation](https://webpack.js.org/configuration/resolve/#resolvefallback)
- [Webpack ProvidePlugin Documentation](https://webpack.js.org/plugins/provide-plugin/)
- [crypto-browserify on npm](https://www.npmjs.com/package/crypto-browserify)
- [Chrome Extension Manifest V3](https://developer.chrome.com/docs/extensions/mv3/)

---

## Conclusion

Building a Chrome extension with Node.js crypto libraries requires careful polyfill configuration. The key is understanding that webpack needs explicit instructions for replacing Node.js modules with browser-compatible alternatives. When standard imports fail, inlining critical functionality provides a reliable fallback.

**Final Result:** A fully functional Solana wallet Chrome extension with proper BIP39/BIP44 key derivation, compatible with Phantom and other standard Solana wallets.

---

**Author:** ZAP Wallet Team  
**Date:** October 9, 2025  
**Version:** 1.0.0

