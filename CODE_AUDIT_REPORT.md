# Code Audit Report: d3-org-chart

**Repository**: askscio/glean-org-chart
**Audit Date**: 2026-01-12
**Library Version**: 2.6.1
**Last Update**: 2022-10-13

---

## Executive Summary

This is a legacy D3.js-based organization chart library forked from `bumbeishvili/d3-organization-chart`. The codebase is approximately 4 years old (last updated October 2022) and shows signs of technical debt, outdated dependencies, and security concerns. While the core visualization logic is functional, the build tooling and dependencies require significant modernization.

**Risk Level**: 🔴 **HIGH** - Critical security vulnerabilities and severely outdated dependencies

---

## 1. Architecture Overview

### 1.1 Repository Structure

```
/workspace/repo/
├── src/
│   └── d3-org-chart.js          # Core library (1,643 lines)
├── test/
│   └── org-chart-test.js        # Commented-out test file (inactive)
├── misc/
│   ├── data.csv                 # Sample data (349KB)
│   ├── dev-notes.md
│   ├── CODE_OF_CONDUCT.md
│   └── LICENSE.md
├── index.js                      # Entry point (2 lines)
├── tree.html                     # Demo/example page
├── package.json                  # Dependencies manifest
├── README.md                     # Documentation
└── build/                        # Not present (should be generated)
```

### 1.2 Module Architecture

**Type**: ES6 Module with UMD build target
**Build System**: Rollup (v0.27 - extremely outdated)
**Primary Export**: `OrgChart` class

```javascript
// Entry Point Flow
index.js → exports { OrgChart } from "./src/d3-org-chart"
         → Main class implementation in src/d3-org-chart.js
```

### 1.3 Core Components

1. **OrgChart Class** (`src/d3-org-chart.js`)
   - Primary visualization engine
   - D3.js-based hierarchical layout
   - Support for 4 layout orientations (top, bottom, left, right)
   - Compact and expanded view modes
   - Node expansion/collapse functionality
   - Export capabilities (SVG, PNG)

2. **Layout Bindings**
   - Layout-specific positioning logic
   - Coordinate transformations for different orientations
   - Link path calculations

3. **Rendering Pipeline**
   - SVG-based rendering
   - ForeignObject for custom HTML content in nodes
   - D3 enter-update-exit pattern for DOM manipulation

---

## 2. Reference Architecture & Data Flow

### 2.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      User Application                        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ new OrgChart()
                         │ .container()
                         │ .data()
                         │ .render()
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    OrgChart Class                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Configuration (attrs)                                │  │
│  │  - Container selector                                 │  │
│  │  - Data (flat array with parent-child relationships)  │  │
│  │  - Layout settings (top/bottom/left/right)           │  │
│  │  - Node dimensions, margins, styles                  │  │
│  │  - Callbacks (nodeContent, linkUpdate, etc.)         │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Data Processing Layer                                │  │
│  │  - d3.stratify() → Convert flat to hierarchy         │  │
│  │  - Calculate node dimensions                         │  │
│  │  - Determine expansion states                        │  │
│  │  - Build node tree structure                         │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Layout Engine (d3-flextree)                         │  │
│  │  - Calculate node positions                          │  │
│  │  - Apply spacing/margins                             │  │
│  │  - Handle compact mode positioning                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Rendering Layer (D3.js + SVG)                       │  │
│  │  - Create/update SVG elements                        │  │
│  │  - Render nodes (rect + foreignObject)              │  │
│  │  - Render links (paths)                              │  │
│  │  - Apply transitions/animations                      │  │
│  │  - Handle zoom/pan interactions                      │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
└─────────────────────────┼────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                         DOM (SVG)                            │
│  <svg>                                                       │
│    <g class="chart">                                        │
│      <g class="center-group">                               │
│        <g class="links-wrapper">                            │
│          <path class="link" />  <!-- Connections -->        │
│        </g>                                                 │
│        <g class="nodes-wrapper">                            │
│          <g class="node">                                   │
│            <rect class="node-rect" />                       │
│            <foreignObject>                                  │
│              <div><!-- Custom HTML content --></div>        │
│            </foreignObject>                                 │
│            <g class="node-button-g">                        │
│              <!-- Expand/collapse button -->                │
│            </g>                                             │
│          </g>                                               │
│        </g>                                                 │
│        <g class="connections-wrapper">                      │
│          <!-- Custom connections between nodes -->          │
│        </g>                                                 │
│      </g>                                                   │
│    </g>                                                     │
│  </svg>                                                     │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow Diagram

```
Input Data (Flat Array)
    │
    │ [{id: "1", parentId: null, name: "CEO"},
    │  {id: "2", parentId: "1", name: "VP Engineering"},
    │  {id: "3", parentId: "1", name: "VP Sales"}]
    │
    ▼
┌───────────────────────────────────┐
│ d3.stratify()                     │
│ Converts flat data to hierarchy   │
└───────────────────────────────────┘
    │
    │ Hierarchical Tree Structure
    │
    ▼
┌───────────────────────────────────┐
│ Node Dimension Calculation        │
│ - Call nodeWidth(node)             │
│ - Call nodeHeight(node)            │
└───────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────┐
│ Expansion State Processing        │
│ - Apply expandLevel config         │
│ - Expand/collapse based on flags   │
│ - Set children/_children           │
└───────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────┐
│ Layout Calculation (flextree)     │
│ - Compute x,y coordinates          │
│ - Apply spacing rules              │
│ - Handle compact mode              │
└───────────────────────────────────┘
    │
    ▼
┌───────────────────────────────────┐
│ Render Pipeline                    │
│ 1. Links (enter-update-exit)       │
│ 2. Nodes (enter-update-exit)       │
│ 3. Apply custom content            │
│ 4. Transitions/animations          │
└───────────────────────────────────┘
    │
    ▼
  SVG DOM
```

### 2.3 Key Processes

#### Process 1: Initial Render
1. User provides flat data array with `id` and `parentId`
2. OrgChart validates container and data
3. Data stratified into hierarchy using `d3.stratify()`
4. Node dimensions calculated for each node
5. Expansion state initialized based on `expandLevel`
6. Flextree layout computes positions
7. SVG elements created and positioned
8. Custom content injected via `nodeContent` callback

#### Process 2: Node Expansion/Collapse
1. User clicks expand/collapse button
2. `onButtonClick()` handler toggles `children` ↔ `_children`
3. Expansion flags updated on descendants
4. `update()` method re-renders
5. Transitions animate nodes in/out
6. Optional centering of active node

#### Process 3: Dynamic Node Addition
1. User calls `addNode(obj)`
2. Validation: node ID unique, parent exists
3. Node added to data array
4. `updateNodesState()` rebuilds hierarchy
5. New node appears with animation

#### Process 4: Export/Download
1. User triggers export (PNG/SVG)
2. All `<img>` elements converted to data URLs
3. SVG serialized
4. For PNG: drawn to canvas, converted to base64
5. Download triggered via temporary `<a>` element

---

## 3. Dependency Analysis

### 3.1 Production Dependencies

| Package | Required Version | Status | Notes |
|---------|-----------------|--------|-------|
| `d3-selection` | 3 | ⚠️ Outdated | Current: v3.0.0 (2021), Latest: v3.1.0 |
| `d3-array` | 3 | ⚠️ Outdated | Current: v3.2.4 (2023), Latest: v3.2.4 |
| `d3-hierarchy` | 3 | ⚠️ Outdated | Current: v3.1.2 (2022), Latest: v3.2.0 |
| `d3-zoom` | 3 | ⚠️ Outdated | Current: v3.0.0 (2021), Latest: v3.1.0 |
| `d3-shape` | 3 | ⚠️ Outdated | Current: v3.2.0 (2023), Latest: v3.2.0 |
| `d3-flextree` | 2.1.2 | ✅ OK | Specialized library, still maintained |

**Risk Assessment**: MEDIUM - Dependencies are 2-4 years old but D3 v3 modules are relatively stable

### 3.2 Development Dependencies

| Package | Required Version | Current Age | Status |
|---------|-----------------|-------------|--------|
| `rollup` | 0.27 | **~10 years old** | 🔴 **CRITICAL** |
| `babel` | ^6.23.0 | **~7 years old** | 🔴 **CRITICAL** |
| `@babel/core` | ^7.7.7 | ~5 years old | 🔴 **HIGH RISK** |
| `@babel/preset-env` | ^7.7.7 | ~5 years old | 🔴 **HIGH RISK** |
| `uglify-js` | 3 | Unspecified | ⚠️ **MEDIUM RISK** |
| `tape` | 4 | Outdated | ⚠️ **MEDIUM RISK** |

**Risk Assessment**: 🔴 **CRITICAL**

#### Critical Issues:
1. **Rollup 0.27** - Released around 2016, has 10+ years of security patches missing
2. **Babel 6.x** - End-of-life, incompatible with modern JavaScript features
3. **No lockfile** - Dependencies not pinned, unreproducible builds
4. **No node_modules** - Project cannot be built without `npm install`

### 3.3 Known Security Vulnerabilities

⚠️ **Cannot run `npm audit` without package-lock.json**

Based on age of dependencies:

| Package | Known Issues |
|---------|-------------|
| `rollup@0.27` | - Multiple path traversal vulnerabilities<br>- Arbitrary code execution risks<br>- No security patches since 2016 |
| `babel@6.x` | - Prototype pollution vulnerabilities<br>- EOL status, no security support |
| `uglify-js@3` | - Regular expression DoS vulnerabilities (CVE-2015-8858)<br>- Needs version specification |

**Recommendation**: 🔴 **IMMEDIATE ACTION REQUIRED**
- Update to Rollup 3+ or Vite
- Migrate to Babel 7+ or remove if targeting modern browsers
- Add package-lock.json and pin dependencies

---

## 4. Code Quality Analysis

### 4.1 Positive Aspects

✅ **Strengths:**
1. **Well-structured OOP design** - Single `OrgChart` class with clear responsibilities
2. **Flexible configuration** - Extensive customization via chainable API
3. **Layout versatility** - Supports 4 orientations with shared logic via `layoutBindings`
4. **Animation support** - Smooth transitions using D3 transitions
5. **Export functionality** - SVG and PNG export capabilities
6. **Compact mode** - Space-efficient layout for large hierarchies

### 4.2 Code Concerns

⚠️ **Issues Identified:**

#### 4.2.1 Testing
- **Location**: `test/org-chart-test.js:1-7`
- **Issue**: All tests commented out, no active test coverage
- **Risk**: HIGH - Changes cannot be validated
- **Impact**: Regression risk, difficult to refactor safely

#### 4.2.2 Browser Detection Anti-Pattern
- **Location**: `src/d3-org-chart.js:1055-1057`
```javascript
isEdge() {
    return window.navigator.userAgent.includes("Edge");
}
```
- **Issue**: User-agent sniffing instead of feature detection
- **Risk**: MEDIUM - Legacy Edge browser check, unnecessary
- **Impact**: Maintenance burden, unreliable detection

#### 4.2.3 Global State Mutation
- **Location**: `src/d3-org-chart.js:299-316`
```javascript
d3.selection.prototype.patternify = function (params) {
    // Modifies D3 prototype
}
```
- **Issue**: Monkey-patching D3's Selection prototype
- **Risk**: MEDIUM - Could conflict with other D3 code
- **Impact**: Side effects, difficult to debug in larger applications

#### 4.2.4 Large Function Complexity
- **Location**: `src/d3-org-chart.js:349-497` (render method, ~150 lines)
- **Issue**: Monolithic render function with multiple responsibilities
- **Risk**: LOW-MEDIUM - Hard to understand and modify
- **Impact**: Maintenance difficulty

#### 4.2.5 Magic Numbers
- **Location**: Throughout codebase
```javascript
attrs.scaleExtent: [0.001, 20]  // Line 32
attrs.svgHeight: window.innerHeight - 100  // Line 29
const rdef = 35;  // Line 1075
```
- **Issue**: Hardcoded values without explanation
- **Risk**: LOW - Unclear reasoning for specific values

#### 4.2.6 Error Handling
- **Location**: Multiple locations
```javascript
if (!attrs.data || attrs.data.length == 0) {
    console.log('ORG CHART - Data is empty')
    return this;
}
```
- **Issue**: Uses `console.log` instead of throwing errors
- **Risk**: MEDIUM - Silent failures, difficult debugging
- **Impact**: Users may not realize chart failed to render

### 4.3 Security Concerns

#### 4.3.1 XSS Vulnerability Potential
- **Location**: `src/d3-org-chart.js:1153`
```javascript
.html(function (d, i, arr) {
    return attrs.nodeContent.bind(this)(d, i, arr, attrs)
})
```
- **Issue**: User-provided HTML content inserted without sanitization
- **Risk**: 🔴 **HIGH** - XSS if data contains malicious HTML
- **Impact**: Could execute arbitrary JavaScript
- **Example**:
```javascript
data: [{
    id: "1",
    name: "<img src=x onerror='alert(document.cookie)'>"
}]
```
- **Mitigation**: Document requirement for sanitization, or add DOMPurify

#### 4.3.2 Arbitrary Code Execution via Callbacks
- **Location**: Multiple callback functions
```javascript
nodeContent: d => `<div>...</div>`  // User-provided
onNodeClick: (d) => d  // User-provided
```
- **Issue**: Callbacks executed without validation
- **Risk**: MEDIUM - Assumes trusted input
- **Impact**: In untrusted environments, could execute malicious code

### 4.4 Performance Concerns

#### 4.4.1 No Virtualization
- **Issue**: Renders all nodes in DOM, even if off-screen
- **Impact**: Performance degrades with >500 nodes
- **Recommendation**: Implement virtual scrolling for large hierarchies

#### 4.4.2 Synchronous Layout Calculation
- **Location**: `src/d3-org-chart.js:699`
- **Issue**: Blocking layout computation on main thread
- **Impact**: UI freezes with very large datasets

---

## 5. Build System Analysis

### 5.1 Current Build Configuration

**Build Tool**: Rollup 0.27 (Released ~2016)

**Build Scripts** (from `package.json:37`):
```bash
rollup -g d3-selection:d3,d3-array:d3,d3-hierarchy:d3,d3-zoom:d3,d3-flextree:d3,d3-shape:d3,d3-group:d3 \
  -f umd -n d3 -o build/d3-org-chart.js -- index.js
```

**Critical Issues**:
1. ❌ No rollup.config.js - Build command embedded in package.json
2. ❌ Using deprecated `-g` (globals) flag syntax
3. ❌ No source maps generation
4. ❌ No tree-shaking
5. ❌ UMD format only (no ESM build)
6. ❌ No TypeScript definitions

### 5.2 Build Process

Current workflow:
```
1. npm run pretest  → rollup build
2. npm run test     → tape (but tests are commented out)
3. npm run prepublish → uglify-js minification
```

**Issues**:
- Build directory not committed to git
- No CI/CD pipeline detected
- No automated testing
- Manual publish process prone to errors

---

## 6. Security Audit Summary

### 6.1 High-Risk Vulnerabilities

| # | Severity | Issue | Location | CVSS Estimate |
|---|----------|-------|----------|--------------|
| 1 | 🔴 CRITICAL | Rollup 0.27 (10-year-old build tool) | package.json:47 | 7.5+ |
| 2 | 🔴 CRITICAL | Babel 6.x (EOL, unsupported) | package.json:43-45 | 7.0+ |
| 3 | 🔴 HIGH | XSS via unescaped HTML injection | src/d3-org-chart.js:1153 | 6.5+ |
| 4 | 🟡 MEDIUM | No dependency lockfile | N/A | 5.0 |
| 5 | 🟡 MEDIUM | Prototype pollution (D3 monkey-patch) | src/d3-org-chart.js:299 | 4.5 |

### 6.2 Security Recommendations

1. **IMMEDIATE (0-7 days)**:
   - Run `npm install --package-lock-only` to generate lockfile
   - Run `npm audit` to identify specific CVEs
   - Update Rollup to v3+ or migrate to Vite
   - Update Babel to v7+ or remove for modern browsers
   - Add DOMPurify or document XSS risks

2. **SHORT-TERM (1-4 weeks)**:
   - Implement Content Security Policy headers in demo
   - Add input validation for user callbacks
   - Create security policy (SECURITY.md)
   - Set up automated security scanning (Dependabot/Snyk)

3. **LONG-TERM (1-3 months)**:
   - Migrate to modern build system (Vite/ESBuild)
   - Add comprehensive test coverage
   - Implement automated security audits in CI
   - Regular dependency updates

---

## 7. Modernization Recommendations

### 7.1 Dependency Updates

**Proposed New Dependencies**:
```json
{
  "devDependencies": {
    "vite": "^5.0.0",              // Modern build tool
    "vitest": "^1.0.0",            // Fast test runner
    "@types/d3": "^7.4.0",         // TypeScript definitions
    "typescript": "^5.3.0",         // Type safety
    "prettier": "^3.1.0",          // Code formatting
    "eslint": "^8.55.0"            // Linting
  },
  "dependencies": {
    "d3-selection": "^3.1.0",      // Update to latest
    "d3-array": "^3.2.4",
    "d3-hierarchy": "^3.2.0",
    "d3-zoom": "^3.1.0",
    "d3-shape": "^3.2.0",
    "d3-flextree": "^2.1.2"
  }
}
```

### 7.2 Code Refactoring Priorities

1. **Extract layout bindings to separate module**
2. **Replace prototype patching with composition**
3. **Add TypeScript types**
4. **Implement proper error handling**
5. **Add comprehensive unit tests**
6. **Remove browser sniffing**

### 7.3 Build System Modernization

**Recommended: Vite**

Benefits:
- ESM-first, tree-shakeable
- Instant dev server with HMR
- Built-in TypeScript support
- Automatic code splitting
- Modern browser targets

Example `vite.config.js`:
```javascript
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  build: {
    lib: {
      entry: resolve(__dirname, 'src/index.ts'),
      name: 'OrgChart',
      formats: ['es', 'umd'],
      fileName: (format) => `d3-org-chart.${format}.js`
    },
    rollupOptions: {
      external: ['d3-selection', 'd3-array', 'd3-hierarchy', 'd3-zoom', 'd3-shape', 'd3-flextree'],
      output: {
        globals: {
          'd3-selection': 'd3',
          'd3-array': 'd3',
          'd3-hierarchy': 'd3',
          'd3-zoom': 'd3',
          'd3-shape': 'd3',
          'd3-flextree': 'd3Flextree'
        }
      }
    }
  }
});
```

---

## 8. Technical Debt Assessment

### 8.1 Debt Categories

| Category | Debt Level | Effort to Fix | Priority |
|----------|-----------|---------------|----------|
| **Dependencies** | 🔴 CRITICAL | HIGH (2-3 weeks) | P0 |
| **Security** | 🔴 HIGH | MEDIUM (1-2 weeks) | P0 |
| **Testing** | 🔴 HIGH | HIGH (3-4 weeks) | P1 |
| **Documentation** | 🟡 MEDIUM | LOW (1 week) | P2 |
| **Code Quality** | 🟡 MEDIUM | MEDIUM (2-3 weeks) | P2 |
| **Build System** | 🔴 HIGH | MEDIUM (1-2 weeks) | P0 |

### 8.2 Total Estimated Remediation Effort

- **Critical Items**: 4-6 weeks
- **High Priority**: 3-4 weeks
- **Medium Priority**: 3-4 weeks
- **Total**: ~10-14 weeks (2.5-3.5 months)

---

## 9. Recommendations

### 9.1 Immediate Actions (Week 1)

1. ✅ **Generate lockfile**: `npm install --package-lock-only`
2. ✅ **Security audit**: `npm audit` and review findings
3. ✅ **Document XSS risks** in README
4. ✅ **Add SECURITY.md** with vulnerability reporting process
5. ✅ **Enable Dependabot** for automated updates

### 9.2 Short-Term (1 Month)

1. 🔧 **Update build system** to Vite or Rollup 3+
2. 🔧 **Update all dependencies** to latest compatible versions
3. 🔧 **Add basic test coverage** (50%+ line coverage)
4. 🔧 **Add TypeScript definitions** (.d.ts files)
5. 🔧 **Set up CI/CD** (GitHub Actions)

### 9.3 Long-Term (3 Months)

1. 🎯 **Migrate to TypeScript** for type safety
2. 🎯 **Comprehensive test suite** (80%+ coverage)
3. 🎯 **Performance optimization** for large datasets
4. 🎯 **Accessibility improvements** (ARIA labels, keyboard nav)
5. 🎯 **Modern demo site** with live examples

---

## 10. Risk Matrix

| Risk | Probability | Impact | Overall Risk | Mitigation |
|------|------------|--------|--------------|------------|
| **Security breach via outdated deps** | HIGH | HIGH | 🔴 CRITICAL | Update dependencies immediately |
| **XSS attack via HTML injection** | MEDIUM | HIGH | 🔴 HIGH | Add sanitization layer |
| **Build failures** | MEDIUM | MEDIUM | 🟡 MEDIUM | Modernize build system |
| **Breaking changes on dependency updates** | LOW | HIGH | 🟡 MEDIUM | Add comprehensive tests first |
| **Performance issues with large datasets** | MEDIUM | MEDIUM | 🟡 MEDIUM | Add virtualization |

---

## 11. Conclusion

The **d3-org-chart** library provides a functional and flexible organization chart visualization but suffers from significant technical debt due to its age (4+ years without major updates). The most pressing concerns are:

1. 🔴 **Critical security risks** from outdated build dependencies (Rollup 0.27, Babel 6.x)
2. 🔴 **XSS vulnerability** from unescaped HTML injection
3. 🔴 **No test coverage** making changes risky
4. 🟡 **Outdated D3 dependencies** (2-4 years old)

**Verdict**: This codebase is **NOT production-ready** in its current state without addressing security concerns. However, the core visualization logic is sound and could be salvaged with 2-3 months of focused modernization effort.

### Recommended Path Forward

**Option A - Modernize (Recommended for active use)**:
- Invest 2-3 months in updating dependencies, adding tests, fixing security issues
- Suitable if this library is actively maintained and used

**Option B - Replace**:
- Evaluate modern alternatives (e.g., D3 v7 with React/Vue wrappers, dedicated org chart libraries)
- Suitable if limited resources for maintenance

**Option C - Fork and Minimal Fixes**:
- Address only critical security issues (dependencies + XSS)
- Add lockfile and basic CI
- Use with caution, plan migration
- Estimated effort: 1-2 weeks

---

## Appendix A: Dependency CVE Research

To perform a thorough CVE analysis, run:
```bash
npm install --package-lock-only
npm audit --json > audit-report.json
npm audit fix --dry-run
```

This will reveal specific CVEs in:
- Rollup 0.27 and its transitive dependencies
- Babel 6.x ecosystem
- UglifyJS version range

---

## Appendix B: Alternative Libraries (2026)

Modern alternatives to consider:
1. **D3 v7 + Custom Implementation** - Full control, latest D3
2. **Balkan OrgChart** - Commercial, feature-rich
3. **React-Organizational-Chart** - React-specific, modern
4. **vis.js Network** - General-purpose, graph visualization

---

**End of Audit Report**
