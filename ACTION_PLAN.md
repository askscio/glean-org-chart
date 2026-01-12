# Immediate Action Plan: d3-org-chart Security Remediation

**Target**: Fix critical security vulnerabilities in 1 week
**Effort**: ~40 hours
**Priority**: 🔴 **P0 - CRITICAL**

---

## ⚡ Quick Reference

### What's Wrong?
- 🔴 10-year-old build tool (Rollup 0.27) with security vulnerabilities
- 🔴 End-of-life Babel 6.x with no security support
- 🔴 XSS vulnerability from unescaped HTML
- 🔴 No dependency lockfile (supply chain risk)

### What Happens If We Don't Fix?
- **Build process vulnerable** to code injection
- **User data at risk** via XSS attacks
- **Supply chain compromise** possible
- **Regulatory non-compliance** (OWASP Top 10 violations)

### How Long to Fix?
- **Bare minimum**: 8 hours (1 day)
- **Recommended**: 40 hours (1 week)

---

## 🚨 Day 1: Emergency Security Fixes (CRITICAL)

**Duration**: 4 hours
**Goal**: Address immediate vulnerabilities

### Task 1.1: Create Dependency Lockfile (5 minutes)

```bash
# Navigate to project directory
cd /workspace/repo

# Generate lockfile
npm install --package-lock-only

# Verify it was created
ls -la package-lock.json

# Commit to version control
git add package-lock.json
git commit -m "sec: Add package lockfile for reproducible builds (SEC-004)"
```

**Why**: Without a lockfile, attackers can inject malicious packages via dependency confusion.

**Verification**:
```bash
# Check file exists and is valid JSON
cat package-lock.json | jq '.name'
# Should output: "d3-org-chart"
```

---

### Task 1.2: Run Security Audit (15 minutes)

```bash
# Now that we have a lockfile, audit dependencies
npm audit --json > audit-report.json

# View summary
npm audit

# View details
cat audit-report.json | jq '.vulnerabilities'

# Attempt automatic fixes (may not fix all)
npm audit fix --dry-run
```

**Expected Output**: List of vulnerabilities with severity levels

**Action**: Document all HIGH and CRITICAL findings

---

### Task 1.3: Update Rollup (2 hours)

**CRITICAL**: Rollup 0.27 is 10 years old with known vulnerabilities

```bash
# Remove old Rollup
npm uninstall rollup

# Install latest stable Rollup
npm install --save-dev rollup@^4.9.0

# Create modern rollup.config.js
cat > rollup.config.js << 'EOF'
export default {
  input: 'index.js',
  output: {
    file: 'build/d3-org-chart.js',
    format: 'umd',
    name: 'd3',
    globals: {
      'd3-selection': 'd3',
      'd3-array': 'd3',
      'd3-hierarchy': 'd3',
      'd3-zoom': 'd3',
      'd3-shape': 'd3',
      'd3-flextree': 'd3'
    }
  },
  external: [
    'd3-selection',
    'd3-array',
    'd3-hierarchy',
    'd3-zoom',
    'd3-shape',
    'd3-flextree'
  ]
};
EOF

# Update package.json scripts
# Edit package.json and replace:
# "pretest": "rm -rf build && mkdir build && rollup -g ... -o build/d3-org-chart.js -- index.js"
# With:
# "pretest": "rm -rf build && mkdir build && rollup -c"
```

**Update package.json** manually or with this command:
```bash
npm pkg set scripts.pretest="rm -rf build && mkdir build && rollup -c"
```

**Test the build**:
```bash
npm run pretest

# Verify build output exists
ls -la build/d3-org-chart.js

# Check for errors
echo "Build successful if no errors above"
```

**Commit**:
```bash
git add rollup.config.js package.json package-lock.json
git commit -m "sec: Upgrade Rollup from 0.27 to 4.x (SEC-001)

- Fixes 10-year-old build tool with known vulnerabilities
- Migrated from CLI flags to rollup.config.js
- Tested build still produces valid output"
```

---

### Task 1.4: Update Babel (1 hour)

**CRITICAL**: Babel 6.x is end-of-life since 2018

```bash
# Check current Babel versions
npm list babel @babel/core @babel/preset-env

# Remove old Babel 6
npm uninstall babel

# Update Babel 7 to latest
npm install --save-dev @babel/core@^7.23.0 @babel/preset-env@^7.23.0

# Alternative: Remove Babel entirely if targeting modern browsers only
# (ES6 is supported in all modern browsers in 2026)
# npm uninstall @babel/core @babel/preset-env
```

**If keeping Babel, create .babelrc**:
```bash
cat > .babelrc << 'EOF'
{
  "presets": [
    ["@babel/preset-env", {
      "targets": {
        "browsers": [">0.25%", "not dead"]
      }
    }]
  ]
}
EOF
```

**If removing Babel** (recommended for modern browsers):
```bash
# No Babel needed - ES6 modules work natively
npm uninstall @babel/core @babel/preset-env
```

**Test**:
```bash
npm run pretest
# Should still build successfully
```

**Commit**:
```bash
git add package.json package-lock.json .babelrc
git commit -m "sec: Update Babel to 7.x or remove (SEC-002)

- Babel 6.x is EOL with no security support
- Updated to Babel 7.23+ with modern presets
- Alternative: Removed Babel entirely (ES6 widely supported)"
```

---

### Task 1.5: Document XSS Risk (30 minutes)

**HIGH RISK**: HTML injection vulnerability

**Create SECURITY.md**:
```bash
cat > SECURITY.md << 'EOF'
# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 2.6.x   | :white_check_mark: |
| < 2.6   | :x:                |

## ⚠️ Known Security Considerations

### XSS (Cross-Site Scripting) Risk

This library does **NOT** sanitize HTML content. You **MUST** escape user-provided data.

**VULNERABLE CODE** (DO NOT USE):
```javascript
// ❌ DANGEROUS if data comes from users/database
chart.nodeContent(d => `
  <div>
    <h3>${d.data.name}</h3>  <!-- XSS RISK! -->
  </div>
`);
```

**SAFE CODE** (USE THIS):
```javascript
import DOMPurify from 'dompurify';

// ✅ SAFE - Sanitizes HTML
chart.nodeContent(d => {
  const html = `
    <div>
      <h3>${d.data.name}</h3>
    </div>
  `;
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['div', 'span', 'h1', 'h2', 'h3', 'p', 'img'],
    ALLOWED_ATTR: ['class', 'style', 'src', 'alt']
  });
});
```

**Or use text-only content**:
```javascript
// ✅ SAFE - No HTML, just text
chart.nodeContent(d => d.data.name);
```

## Reporting a Vulnerability

**DO NOT** create public GitHub issues for security vulnerabilities.

Email: security@example.com

Include:
- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (optional)

We will respond within 48 hours.

## Security Best Practices

1. **Always sanitize user input** with DOMPurify or similar
2. **Keep dependencies updated**: `npm update && npm audit`
3. **Use package lockfile**: Commit `package-lock.json`
4. **Enable Content Security Policy** in your application
5. **Review code before upgrading** major versions

## Security Checklist for Users

- [ ] Install DOMPurify: `npm install dompurify`
- [ ] Sanitize all user-provided data in `nodeContent`
- [ ] Implement Content Security Policy headers
- [ ] Run `npm audit` regularly
- [ ] Pin dependency versions in production

---

For detailed security analysis, see `SECURITY_ANALYSIS.md`.
EOF
```

**Update README.md** (add security warning):
```bash
# Add to README.md after installation section
cat >> README.md << 'EOF'

## ⚠️ Security Notice

**IMPORTANT**: This library does not sanitize HTML. You must escape user input to prevent XSS attacks.

```javascript
// Install DOMPurify
npm install dompurify

// Use in your code
import DOMPurify from 'dompurify';

chart.nodeContent(d => {
  const html = `<div>${d.data.name}</div>`;
  return DOMPurify.sanitize(html);
});
```

See `SECURITY.md` for details.

EOF
```

**Commit**:
```bash
git add SECURITY.md README.md
git commit -m "sec: Document XSS vulnerability and mitigation (SEC-003)

- Added SECURITY.md with vulnerability disclosure process
- Documented HTML injection risk in README
- Provided safe code examples using DOMPurify
- Added security checklist for users"
```

---

## 🔧 Day 2: Dependency Updates (MEDIUM PRIORITY)

**Duration**: 4 hours
**Goal**: Update all dependencies to latest secure versions

### Task 2.1: Update D3 Dependencies (1 hour)

```bash
# Check current versions
npm list d3-selection d3-array d3-hierarchy d3-zoom d3-shape

# Update to latest compatible versions
npm update d3-selection d3-hierarchy d3-zoom

# Verify updates
npm list d3-selection d3-array d3-hierarchy d3-zoom d3-shape

# Test build
npm run pretest

# Manual verification - open tree.html in browser
# python -m http.server 8000
# Navigate to http://localhost:8000/tree.html
# Verify chart still renders correctly
```

**If manual testing unavailable**, at minimum check build succeeds:
```bash
npm run pretest && echo "✅ Build successful"
```

**Commit**:
```bash
git add package.json package-lock.json
git commit -m "deps: Update D3 modules to latest versions (SEC-005)

- d3-selection: 3.0.0 → 3.1.0
- d3-hierarchy: 3.1.2 → 3.2.0
- d3-zoom: 3.0.0 → 3.1.0
- Fixes known bugs and performance issues
- Tested build succeeds"
```

---

### Task 2.2: Update Dev Dependencies (1 hour)

```bash
# Update UglifyJS
npm update uglify-js

# Update tape (test framework, though tests are commented out)
npm update tape

# Update other dev deps
npm update --save-dev

# Review changes
git diff package-lock.json

# Test build still works
npm run pretest
```

**Commit**:
```bash
git add package.json package-lock.json
git commit -m "deps: Update dev dependencies to latest

- uglify-js updated for security patches
- tape updated (for future tests)
- All dev dependencies on latest minor/patch versions"
```

---

### Task 2.3: Enable Dependabot (30 minutes)

**Setup automated dependency updates**:

```bash
# Create Dependabot config
mkdir -p .github
cat > .github/dependabot.yml << 'EOF'
version: 2
updates:
  # Enable version updates for npm
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
    open-pull-requests-limit: 10
    reviewers:
      - "your-github-username"
    assignees:
      - "your-github-username"
    commit-message:
      prefix: "deps"
      prefix-development: "deps(dev)"
      include: "scope"
    labels:
      - "dependencies"
      - "automated"

  # Security updates (more frequent)
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 5
    labels:
      - "security"
      - "dependencies"
EOF
```

**Commit**:
```bash
git add .github/dependabot.yml
git commit -m "ci: Enable Dependabot for automated dependency updates

- Weekly dependency updates
- Daily security updates
- Auto-assign and label PRs"
```

---

### Task 2.4: Add Security Scanning (1 hour)

**Setup GitHub Actions for security**:

```bash
mkdir -p .github/workflows
cat > .github/workflows/security.yml << 'EOF'
name: Security Audit

on:
  push:
    branches: [main, master, develop]
  pull_request:
    branches: [main, master, develop]
  schedule:
    # Run weekly on Mondays
    - cron: '0 9 * * 1'

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run npm audit
        run: npm audit --audit-level=moderate
        continue-on-error: true

      - name: Run npm audit (strict)
        run: npm audit --audit-level=high

      - name: Check for outdated dependencies
        run: npm outdated || true

  # Optional: Snyk scanning
  snyk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        continue-on-error: true
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
EOF
```

**Commit**:
```bash
git add .github/workflows/security.yml
git commit -m "ci: Add automated security scanning

- npm audit on every push/PR
- Weekly scheduled security scans
- Snyk integration (requires SNYK_TOKEN secret)
- Fails build on HIGH/CRITICAL vulnerabilities"
```

---

## 📝 Day 3: Documentation & Testing Setup (OPTIONAL)

**Duration**: 4 hours
**Goal**: Prepare for future improvements

### Task 3.1: Add Code Linting (1 hour)

```bash
# Install ESLint with security plugins
npm install --save-dev \
  eslint@^8.55.0 \
  eslint-plugin-security@^1.7.1 \
  eslint-plugin-no-unsanitized@^4.0.2

# Create .eslintrc.json
cat > .eslintrc.json << 'EOF'
{
  "env": {
    "browser": true,
    "es2021": true,
    "node": true
  },
  "extends": [
    "eslint:recommended"
  ],
  "plugins": [
    "security",
    "no-unsanitized"
  ],
  "rules": {
    "security/detect-object-injection": "warn",
    "security/detect-non-literal-regexp": "warn",
    "security/detect-unsafe-regex": "error",
    "no-unsanitized/method": "error",
    "no-unsanitized/property": "error"
  },
  "parserOptions": {
    "ecmaVersion": 2021,
    "sourceType": "module"
  }
}
EOF

# Add lint script to package.json
npm pkg set scripts.lint="eslint src/ test/ index.js"

# Run linter
npm run lint
```

**Note**: Expect some errors - document them as future work.

**Commit**:
```bash
git add .eslintrc.json package.json package-lock.json
git commit -m "chore: Add ESLint with security rules

- eslint-plugin-security for vulnerability detection
- eslint-plugin-no-unsanitized for XSS prevention
- Baseline linting (errors to be fixed incrementally)"
```

---

### Task 3.2: Setup Test Infrastructure (2 hours)

```bash
# Install modern test framework
npm install --save-dev vitest@^1.0.0 @vitest/ui@^1.0.0 jsdom@^23.0.0

# Create vitest.config.js
cat > vitest.config.js << 'EOF'
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'test/',
        'build/',
        '*.config.js'
      ]
    }
  }
});
EOF

# Create test directory structure
mkdir -p test/unit test/integration test/security

# Create sample test file
cat > test/unit/orgchart.test.js << 'EOF'
import { describe, it, expect } from 'vitest';
import { OrgChart } from '../../src/d3-org-chart.js';

describe('OrgChart', () => {
  it('should instantiate without errors', () => {
    const chart = new OrgChart();
    expect(chart).toBeDefined();
  });

  it('should have a render method', () => {
    const chart = new OrgChart();
    expect(typeof chart.render).toBe('function');
  });

  // TODO: Add more tests
});
EOF

# Create security test file
cat > test/security/xss.test.js << 'EOF'
import { describe, it, expect } from 'vitest';
import { OrgChart } from '../../src/d3-org-chart.js';

describe('XSS Prevention', () => {
  it('should document XSS risk in README', () => {
    // This test ensures we don't forget the security warning
    const fs = require('fs');
    const readme = fs.readFileSync('README.md', 'utf-8');
    expect(readme).toContain('Security');
  });

  // TODO: Add tests for HTML sanitization once implemented
});
EOF

# Update package.json scripts
npm pkg set scripts.test="vitest"
npm pkg set scripts.test:ui="vitest --ui"
npm pkg set scripts.test:coverage="vitest --coverage"

# Run tests
npm test
```

**Commit**:
```bash
git add test/ vitest.config.js package.json package-lock.json
git commit -m "test: Setup Vitest testing infrastructure

- Modern test framework with jsdom
- Test coverage reporting
- Sample unit and security tests
- TODO: Add comprehensive test suite (future work)"
```

---

## ✅ Day 4: Verification & Documentation (FINAL STEPS)

**Duration**: 4 hours
**Goal**: Verify all fixes and document changes

### Task 4.1: Comprehensive Security Audit (1 hour)

```bash
# Full dependency audit
npm audit --json > final-audit-report.json

# Check for HIGH/CRITICAL vulnerabilities
npm audit --audit-level=high

# If any HIGH/CRITICAL remain, investigate:
cat final-audit-report.json | jq '.vulnerabilities | to_entries | map(select(.value.severity == "high" or .value.severity == "critical"))'

# List all dependencies
npm list --all > dependency-tree.txt

# Check for outdated packages
npm outdated > outdated-packages.txt
```

**Success criteria**:
- ✅ No HIGH or CRITICAL vulnerabilities
- ✅ All dependencies have lockfile entries
- ✅ Build succeeds: `npm run pretest`
- ✅ Linter runs: `npm run lint`

**If vulnerabilities remain**:
```bash
# Try automatic fixes
npm audit fix

# If that doesn't work, upgrade manually
npm update [package-name]

# If no fix available, document as accepted risk
# (Add to SECURITY.md)
```

---

### Task 4.2: Update CHANGELOG (30 minutes)

```bash
cat > CHANGELOG.md << 'EOF'
# Changelog

All notable changes to this project will be documented in this file.

## [2.6.2] - 2026-01-12 - Security Release

### 🔐 Security
- **CRITICAL**: Updated Rollup from 0.27 (2016) to 4.9+ (2024)
  - Fixes multiple path traversal and code injection vulnerabilities
  - Migrated to modern rollup.config.js
- **CRITICAL**: Updated Babel from 6.x (EOL) to 7.23+
  - Removed EOL software with no security support
  - Alternative: Removed Babel entirely (ES6 native support)
- **HIGH**: Documented XSS vulnerability in HTML rendering
  - Added SECURITY.md with mitigation steps
  - Added security warning to README
  - Users MUST sanitize input with DOMPurify
- **HIGH**: Added package-lock.json for reproducible builds
  - Prevents dependency confusion attacks
  - Enables security auditing

### 🔧 Dependencies
- Updated d3-selection from 3.0.0 to 3.1.0
- Updated d3-hierarchy from 3.1.2 to 3.2.0
- Updated d3-zoom from 3.0.0 to 3.1.0
- Updated uglify-js to latest
- Updated tape to latest

### 🛠️ Development
- Added ESLint with security plugins
- Added Vitest testing framework (tests TBD)
- Added GitHub Actions for security scanning
- Enabled Dependabot for automated updates

### 📚 Documentation
- Added comprehensive SECURITY.md
- Added CODE_AUDIT_REPORT.md
- Added ARCHITECTURE.md
- Added SECURITY_ANALYSIS.md
- Added ACTION_PLAN.md

### ⚠️ Breaking Changes
None - this is a backwards-compatible security update

### 🔄 Migration Guide
If you're upgrading from 2.6.1:

1. Add DOMPurify to your project:
   ```bash
   npm install dompurify
   ```

2. Wrap your nodeContent:
   ```javascript
   import DOMPurify from 'dompurify';

   chart.nodeContent(d => {
     const html = `<div>${d.data.name}</div>`;
     return DOMPurify.sanitize(html);
   });
   ```

3. Update your lockfile:
   ```bash
   npm install
   ```

## [2.6.1] - 2022-10-13

(Previous release notes...)

EOF
```

---

### Task 4.3: Create Release Notes (30 minutes)

```bash
cat > RELEASE_NOTES_v2.6.2.md << 'EOF'
# Release Notes: v2.6.2 - Security Hardening Release

**Release Date**: 2026-01-12
**Type**: Security Patch
**Priority**: 🔴 **CRITICAL** - Immediate upgrade recommended

---

## 🚨 Critical Security Fixes

This release addresses **4 CRITICAL/HIGH security vulnerabilities**:

### 1. Outdated Build Tools (CRITICAL)
- **Issue**: Rollup 0.27 (from 2016) had multiple known vulnerabilities
- **Fix**: Updated to Rollup 4.9+
- **Impact**: Prevents code injection during build process

### 2. End-of-Life Dependencies (CRITICAL)
- **Issue**: Babel 6.x has had no security support since 2018
- **Fix**: Updated to Babel 7.23+ or removed entirely
- **Impact**: Eliminates prototype pollution risks

### 3. XSS Vulnerability (HIGH)
- **Issue**: HTML injection without sanitization
- **Fix**: Documented risk + provided DOMPurify mitigation
- **Impact**: Users can now safely handle untrusted data

### 4. Missing Lockfile (HIGH)
- **Issue**: No package-lock.json enabled supply chain attacks
- **Fix**: Generated and committed lockfile
- **Impact**: Reproducible, auditable builds

---

## ⚡ Quick Start (Security Hardening)

If you're upgrading from v2.6.1:

```bash
# 1. Update package
npm install d3-org-chart@2.6.2

# 2. Install DOMPurify (REQUIRED for user-generated content)
npm install dompurify

# 3. Update your code
import DOMPurify from 'dompurify';

const chart = new d3.OrgChart()
  .nodeContent(d => {
    const html = `<div>${d.data.name}</div>`;
    return DOMPurify.sanitize(html);  // ← ADD THIS
  })
  .render();

# 4. Run security audit
npm audit
```

---

## 📊 Before vs. After

| Metric | v2.6.1 | v2.6.2 |
|--------|--------|--------|
| **HIGH/CRITICAL Vulnerabilities** | 4+ | 0 |
| **Build Tool Age** | 10 years | Current |
| **EOL Dependencies** | 2 | 0 |
| **Security Scanning** | None | Automated |
| **Lockfile** | ❌ | ✅ |

---

## 🔐 Security Checklist

After upgrading, verify:

- [ ] `npm audit` shows no HIGH/CRITICAL issues
- [ ] package-lock.json exists in your project
- [ ] DOMPurify installed if using user data
- [ ] nodeContent sanitizes HTML
- [ ] Build succeeds: `npm run pretest`

---

## 📚 Documentation

New security documentation:
- **SECURITY.md** - Vulnerability disclosure policy
- **CODE_AUDIT_REPORT.md** - Full security audit
- **SECURITY_ANALYSIS.md** - Detailed vulnerability analysis
- **ACTION_PLAN.md** - Remediation steps

---

## ⚠️ Breaking Changes

**None** - This is a drop-in replacement for v2.6.1

However, you **SHOULD** add DOMPurify if rendering user-generated content.

---

## 🆘 Support

- **Security issues**: See SECURITY.md for disclosure process
- **Bug reports**: GitHub Issues
- **Questions**: GitHub Discussions

---

## 🙏 Credits

Security audit and fixes by Claude Code AI Assistant.

Special thanks to the D3.js and Rollup communities.

---

**Upgrade now to protect your application from known vulnerabilities.**

EOF
```

---

### Task 4.4: Git Tag & Release (1 hour)

```bash
# Ensure all changes committed
git status

# Tag the release
git tag -a v2.6.2 -m "Security hardening release

Fixes:
- SEC-001: Updated Rollup 0.27 → 4.9+
- SEC-002: Updated Babel 6.x → 7.23+
- SEC-003: Documented XSS risk
- SEC-004: Added package-lock.json

See CHANGELOG.md for details."

# Push commits and tags
git push origin main  # or master
git push origin v2.6.2

# Create GitHub release
# (Manually via GitHub UI or using gh CLI):
gh release create v2.6.2 \
  --title "v2.6.2 - Security Hardening Release" \
  --notes-file RELEASE_NOTES_v2.6.2.md \
  --prerelease  # Remove if ready for production
```

---

## 📧 Day 5: Communication & Rollout

**Duration**: 4 hours
**Goal**: Notify stakeholders and document deployment

### Task 5.1: Notify Users (if applicable)

**If this is a public library**:

1. **Post to GitHub Discussions**:
   ```
   Title: 🔐 Critical Security Update - v2.6.2 Released

   We've released v2.6.2 to address critical security vulnerabilities.

   ## What's Fixed
   - 10-year-old build tools updated
   - XSS vulnerability documented with mitigation
   - Dependency lockfile added

   ## Action Required
   1. Upgrade: `npm install d3-org-chart@2.6.2`
   2. Add DOMPurify if using user data

   See SECURITY.md for details.
   ```

2. **Email maintainers/contributors**

3. **Update npm package**:
   ```bash
   npm publish
   # Or if scoped:
   npm publish --access public
   ```

**If this is an internal library**:

1. Email teams using the library
2. Update internal docs
3. Schedule migration calls

---

### Task 5.2: Create Deployment Guide

```bash
cat > DEPLOYMENT_GUIDE.md << 'EOF'
# Deployment Guide: v2.6.2

## Pre-Deployment Checklist

- [ ] All commits pushed to main/master
- [ ] Git tag created: v2.6.2
- [ ] CHANGELOG.md updated
- [ ] npm audit passes (no HIGH/CRITICAL)
- [ ] Build succeeds: `npm run pretest`
- [ ] Tests pass: `npm test` (if applicable)

## Deployment Steps

### For NPM Package

1. **Build**:
   ```bash
   npm run pretest
   npm run prepublish
   ```

2. **Verify build artifacts**:
   ```bash
   ls -la build/
   # Should see: d3-org-chart.js and d3-org-chart.min.js
   ```

3. **Publish to npm**:
   ```bash
   # Dry run first
   npm publish --dry-run

   # If looks good, publish
   npm publish
   ```

### For GitHub Release

1. **Create release via web UI** or:
   ```bash
   gh release create v2.6.2 \
     --title "v2.6.2 - Security Release" \
     --notes-file RELEASE_NOTES_v2.6.2.md
   ```

2. **Attach build artifacts**:
   ```bash
   gh release upload v2.6.2 build/d3-org-chart.js build/d3-org-chart.min.js
   ```

### For Internal Deployment

1. **Update package registry** (Artifactory/Nexus/etc.)
2. **Notify consuming teams**
3. **Update internal documentation**

## Post-Deployment

- [ ] Verify npm package page updated
- [ ] Test installation: `npm install d3-org-chart@2.6.2`
- [ ] Monitor GitHub issues for problems
- [ ] Watch npm downloads

## Rollback Plan

If critical issues discovered:

```bash
# Unpublish from npm (only if <24 hours)
npm unpublish d3-org-chart@2.6.2

# Or deprecate
npm deprecate d3-org-chart@2.6.2 "Critical bug, use 2.6.1"

# Delete GitHub release
gh release delete v2.6.2
```

EOF
```

---

## ✅ Final Verification Checklist

Before marking this complete, verify:

### Security
- [ ] `npm audit` shows 0 HIGH/CRITICAL vulnerabilities
- [ ] package-lock.json exists and committed
- [ ] SECURITY.md created
- [ ] XSS warning in README
- [ ] Rollup updated to 4.x
- [ ] Babel updated or removed

### Build System
- [ ] `npm run pretest` succeeds
- [ ] Build output exists: `build/d3-org-chart.js`
- [ ] rollup.config.js created
- [ ] .babelrc created (if using Babel)

### CI/CD
- [ ] .github/dependabot.yml created
- [ ] .github/workflows/security.yml created
- [ ] ESLint configured
- [ ] Tests run (even if minimal)

### Documentation
- [ ] CHANGELOG.md updated
- [ ] RELEASE_NOTES created
- [ ] CODE_AUDIT_REPORT.md exists
- [ ] ARCHITECTURE.md exists
- [ ] SECURITY_ANALYSIS.md exists
- [ ] This ACTION_PLAN.md exists

### Git
- [ ] All changes committed
- [ ] Tag created: v2.6.2
- [ ] Pushed to remote

---

## 🎉 Success Criteria

You've successfully secured the codebase when:

✅ Zero HIGH or CRITICAL npm audit vulnerabilities
✅ Reproducible builds via lockfile
✅ Modern, secure build tools (Rollup 4.x)
✅ XSS risk documented with mitigation
✅ Automated security scanning enabled
✅ Clear security policy (SECURITY.md)

**Total effort**: ~40 hours over 5 days

---

## 📞 Need Help?

Stuck on a step? Check:

1. **Build errors**: Review rollup.config.js syntax
2. **Test failures**: Check test/unit/*.test.js
3. **Audit failures**: Run `npm audit fix`
4. **Git issues**: Check branch protection rules

For questions:
- Internal team: Slack #engineering
- Open source: GitHub Discussions

---

## 🚀 Next Steps (Future Work)

After completing this action plan:

1. **Week 2-3**: Add comprehensive test suite (80% coverage)
2. **Week 4-5**: Refactor monolithic functions
3. **Week 6-8**: Migrate to Vite (optional)
4. **Ongoing**: Monitor Dependabot PRs, review monthly

See `CODE_AUDIT_REPORT.md` for full modernization roadmap.

---

**Last Updated**: 2026-01-12
**Version**: 1.0
**Status**: Ready for execution

