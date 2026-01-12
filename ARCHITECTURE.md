# Architecture Documentation: d3-org-chart

## Table of Contents
1. [System Overview](#system-overview)
2. [Component Architecture](#component-architecture)
3. [Data Flow](#data-flow)
4. [API Design](#api-design)
5. [Rendering Pipeline](#rendering-pipeline)
6. [State Management](#state-management)

---

## 1. System Overview

### 1.1 Purpose
D3-org-chart is a hierarchical visualization library for creating interactive organizational charts using D3.js. It transforms flat data structures into visual tree layouts with support for expansion/collapse, zooming, panning, and custom styling.

### 1.2 Technology Stack

```
┌─────────────────────────────────────────────┐
│           User Application Layer             │
│    (HTML/CSS/JavaScript/Framework)           │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│          d3-org-chart Library                │
│  ┌────────────────────────────────────────┐ │
│  │     OrgChart Class (ES6)               │ │
│  │  - Configuration Management             │ │
│  │  - Data Processing                     │ │
│  │  - Rendering Orchestration             │ │
│  └────────────────────────────────────────┘ │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│           D3.js Modules (v7)                 │
│  ┌──────────┬──────────┬──────────────────┐ │
│  │d3-select │d3-array  │ d3-hierarchy     │ │
│  │d3-zoom   │d3-shape  │ d3-flextree      │ │
│  └──────────┴──────────┴──────────────────┘ │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│              DOM (SVG + HTML)                │
│  <svg>, <foreignObject>, <div>               │
└─────────────────────────────────────────────┘
```

### 1.3 Core Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `d3-selection` | 3.x | DOM manipulation, event handling |
| `d3-array` | 3.x | Array utilities (min, max, sum) |
| `d3-hierarchy` | 3.x | Stratify flat data into trees |
| `d3-zoom` | 3.x | Zoom and pan interactions |
| `d3-shape` | 3.x | Link path generation |
| `d3-flextree` | 2.1.2 | Flexible tree layout algorithm |

---

## 2. Component Architecture

### 2.1 Class Structure

```javascript
OrgChart
│
├── Constructor
│   ├── Initialize attrs (configuration object)
│   ├── Create dynamic getters/setters
│   └── Register patternify helper
│
├── Public API Methods
│   ├── Configuration Setters
│   │   ├── container(selector)
│   │   ├── data(array)
│   │   ├── nodeWidth(function)
│   │   ├── nodeHeight(function)
│   │   └── ... (50+ configuration options)
│   │
│   ├── Rendering Methods
│   │   ├── render()
│   │   └── updateNodesState()
│   │
│   ├── Data Manipulation
│   │   ├── addNode(obj)
│   │   ├── addNodes(array)
│   │   ├── removeNode(id)
│   │   └── closeNonReportingLineNodes(node)
│   │
│   ├── Node State Control
│   │   ├── setExpanded(id, flag)
│   │   ├── setCentered(id)
│   │   ├── setHighlighted(id)
│   │   ├── setUpToTheRootHighlighted(id)
│   │   ├── clearHighlighting()
│   │   ├── expandAll()
│   │   └── collapseAll()
│   │
│   ├── View Control
│   │   ├── fit(params)
│   │   ├── zoomIn()
│   │   ├── zoomOut()
│   │   ├── fullscreen(elem)
│   │   └── initialZoom(level)
│   │
│   └── Export Methods
│       ├── exportImg(options)
│       ├── exportSvg()
│       └── downloadImage(params)
│
└── Private/Internal Methods
    ├── update(node)
    ├── setLayouts(params)
    ├── collapse(node)
    ├── expand(node)
    ├── expandSomeNodes(node)
    ├── calculateCompactFlexDimensions(root)
    ├── calculateCompactFlexPositions(root)
    ├── diagonal(source, target, mid)
    ├── hdiagonal(source, target, mid)
    ├── zoomed(event, data)
    ├── onButtonClick(event, node)
    ├── restyleForeignObjectElements()
    ├── getNodeChildren(node, store)
    └── getTextWidth(text, options)
```

### 2.2 Configuration Object (attrs)

The `attrs` object serves as the central state container:

```javascript
{
  // Identity & Control
  id: 'ID123456',                    // Unique chart instance ID
  firstDraw: true,                   // First render flag

  // Container & Dimensions
  container: 'body',                 // CSS selector for mount point
  svgWidth: 800,                     // Chart width
  svgHeight: 600,                    // Chart height

  // Data & State
  data: [],                          // Flat array of nodes
  allNodes: [],                      // Hierarchy nodes (computed)
  root: null,                        // Root hierarchy node
  addedNodeIds: Set{},               // Tracked node IDs
  hiddenNodeIds: Set{},              // Hidden node IDs

  // Layout Configuration
  layout: 'top',                     // 'top' | 'bottom' | 'left' | 'right'
  compact: true,                     // Compact mode enabled
  expandLevel: 1,                    // Initial expansion depth

  // Node Dimensions (Functions)
  nodeWidth: d => 250,               // Node width calculator
  nodeHeight: d => 150,              // Node height calculator

  // Spacing (Functions)
  siblingsMargin: d => 20,           // Horizontal spacing
  childrenMargin: d => 90,           // Vertical spacing
  neightbourMargin: (n1,n2) => 80,   // Between subtrees
  compactMarginPair: d => 100,       // Compact node pairs
  compactMarginBetween: d => 20,     // Compact rows
  rootMargin: 40,                    // Root offset

  // Visual Styling
  backgroundColor: 'none',
  nodeDefaultBackground: 'none',
  defaultTextFill: '#2C3E50',
  defaultFont: 'Helvetica',
  linkColor: '#EAEBED',
  highlightedLinkColor: '#343ced',

  // Behavior
  duration: 400,                     // Animation duration (ms)
  setActiveNodeCentered: true,       // Auto-center on click
  scaleExtent: [0.001, 20],          // Zoom limits

  // Zoom State
  lastTransform: { x: 0, y: 0, k: 1 },
  zoomBehavior: null,                // D3 zoom behavior

  // Custom Renderers (Functions)
  nodeContent: d => '...',           // Node HTML generator
  buttonContent: ({node, state}) => '...',
  linkUpdate: (d, i, arr) => {},     // Link styling hook
  nodeUpdate: (d, i, arr) => {},     // Node styling hook
  connectionsUpdate: (d, i, arr) => {},
  defs: (state, connections) => '', // SVG defs

  // Callbacks
  onNodeClick: d => d,               // Click handler
  nodeId: d => d.nodeId || d.id,     // ID accessor
  parentNodeId: d => d.parentNodeId, // Parent ID accessor

  // Layout Bindings
  layoutBindings: {                  // Layout-specific logic
    top: { ... },
    bottom: { ... },
    left: { ... },
    right: { ... }
  },

  // References (Set during render)
  svg: null,                         // D3 selection of <svg>
  chart: null,                       // D3 selection of chart <g>
  centerG: null,                     // Center group
  linksWrapper: null,                // Links container
  nodesWrapper: null,                // Nodes container
  connectionsWrapper: null,          // Custom connections
  defsWrapper: null,                 // SVG definitions

  // Connections
  connections: [],                   // Custom node connections

  // Layout Engine
  flexTreeLayout: null,              // d3-flextree instance

  // Utilities
  ctx: Canvas2DContext,              // For text measurement
  calc: {                            // Computed values
    chartWidth: 800,
    chartHeight: 600,
    centerX: 400,
    centerY: 300
  }
}
```

### 2.3 Layout Bindings Architecture

Each layout orientation (top/bottom/left/right) has binding functions:

```javascript
layoutBindings[layout] = {
  // Node Positioning
  nodeLeftX: node => number,         // Left edge X
  nodeRightX: node => number,        // Right edge X
  nodeTopY: node => number,          // Top edge Y
  nodeBottomY: node => number,       // Bottom edge Y

  // Link Join Points
  nodeJoinX: node => number,         // Where links attach (X)
  nodeJoinY: node => number,         // Where links attach (Y)
  linkJoinX: node => number,         // Child link start (X)
  linkJoinY: node => number,         // Child link start (Y)
  linkX: node => number,             // Link target (X)
  linkY: node => number,             // Link target (Y)
  linkParentX: node => number,       // Parent link end (X)
  linkParentY: node => number,       // Parent link end (Y)

  // Compact Mode
  linkCompactXStart: node => number,
  linkCompactYStart: node => number,
  compactLinkMidX: (node, state) => number,
  compactLinkMidY: (node, state) => number,

  // Button Positioning
  buttonX: node => number,
  buttonY: node => number,

  // Transforms
  centerTransform: ({...}) => string,     // Initial positioning
  zoomTransform: ({...}) => string,       // Zoom positioning
  nodeUpdateTransform: ({...}) => string, // Node positioning

  // Flex Layout
  nodeFlexSize: ({...}) => [width, height],
  compactDimension: {
    sizeColumn: node => number,
    sizeRow: node => number,
    reverse: arr => arr
  },

  // Path Generators
  diagonal: (source, target, mid) => string,
  swap: d => void                    // Coordinate transformation
}
```

---

## 3. Data Flow

### 3.1 Input Data Format

```javascript
// Flat array structure
const data = [
  {
    id: "1",                   // Required: Unique identifier
    parentId: null,            // Required: Parent ID (null for root)
    name: "CEO",               // Optional: Custom fields
    department: "Executive",
    imageUrl: "path/to/img.jpg",

    // Internal state (optional, managed by library)
    _expanded: false,          // Expansion state
    _highlighted: false,       // Highlight flag
    _centered: false,          // Centering request
    _directSubordinates: 0,    // Computed: direct children count
    _totalSubordinates: 0      // Computed: all descendants count
  },
  {
    id: "2",
    parentId: "1",
    name: "VP Engineering"
  },
  {
    id: "3",
    parentId: "1",
    name: "VP Sales"
  }
];
```

### 3.2 Data Transformation Pipeline

```
Input: Flat Array
    │
    ├─► Validation
    │     ├─ Check data exists
    │     ├─ Check unique IDs
    │     └─ Check parent references
    │
    ▼
d3.stratify()
    │
    ├─► Creates Hierarchy
    │     └─ Each node gets:
    │         ├─ .data (original object)
    │         ├─ .parent (parent node)
    │         ├─ .children (array of child nodes)
    │         ├─ .depth (level in tree)
    │         └─ .height (max depth to leaves)
    │
    ▼
Node Enhancement
    │
    ├─► Add Computed Properties
    │     ├─ .width (from nodeWidth function)
    │     ├─ .height (from nodeHeight function)
    │     ├─ ._directSubordinates (count)
    │     └─ ._totalSubordinates (count)
    │
    ▼
Expansion State Processing
    │
    ├─► Based on expandLevel config
    │     ├─ Expand to depth N
    │     ├─ Set .children or ._children
    │     └─ Mark nodes with ._expanded flag
    │
    ▼
Layout Calculation (d3-flextree)
    │
    ├─► Compute Positions
    │     ├─ .x (horizontal position)
    │     ├─ .y (vertical position)
    │     └─ Handle spacing/margins
    │
    ▼
Compact Mode Processing (if enabled)
    │
    ├─► Group Leaf Nodes
    │     ├─ Identify compact pairs
    │     ├─ Calculate flexCompactDim
    │     └─ Adjust positions
    │
    ▼
Coordinate Transformation
    │
    ├─► Layout-specific swap
    │     └─ Adjust for top/bottom/left/right
    │
    ▼
Render to SVG
    │
    └─► Create/Update/Exit Pattern
        ├─ Links (paths)
        ├─ Nodes (rect + foreignObject)
        └─ Buttons (expand/collapse)
```

### 3.3 Update Cycle

```
User Action (click, addNode, etc.)
    │
    ▼
State Mutation
    │
    ├─► Add/Remove from data array
    ├─► Toggle expansion flags
    └─► Set centering/highlighting flags
    │
    ▼
updateNodesState() or update(node)
    │
    ├─► Re-run stratify
    ├─► Re-calculate layout
    └─► Re-compute compact mode
    │
    ▼
D3 Enter-Update-Exit
    │
    ├─► ENTER: New nodes/links
    │     └─ Transition from parent position
    │
    ├─► UPDATE: Existing nodes/links
    │     └─ Transition to new positions
    │
    └─► EXIT: Removed nodes/links
          └─ Transition to parent position
    │
    ▼
Apply Custom Styling
    │
    ├─► Call linkUpdate()
    ├─► Call nodeUpdate()
    └─► Call connectionsUpdate()
    │
    ▼
Handle Centering (if requested)
    │
    └─► fit() to center on node
```

---

## 4. API Design

### 4.1 Fluent Interface Pattern

The library uses method chaining for configuration:

```javascript
const chart = new d3.OrgChart()
  .container('.chart-container')
  .data(employeeData)
  .nodeWidth(d => 220)
  .nodeHeight(d => 85)
  .childrenMargin(d => 50)
  .compact(true)
  .onNodeClick(id => console.log('Clicked:', id))
  .render();
```

Implementation:
```javascript
// Each method returns `this` for chaining
nodeWidth(func) {
  if (!arguments.length) {
    return attrs.nodeWidth;  // Getter
  }
  attrs.nodeWidth = func;    // Setter
  return this;               // Enable chaining
}
```

### 4.2 Configuration Categories

**1. Data & Container**
- `.container(selector)` - Mount point
- `.data(array)` - Flat employee data

**2. Node Dimensions**
- `.nodeWidth(func)` - Width calculator
- `.nodeHeight(func)` - Height calculator

**3. Layout & Spacing**
- `.layout(orientation)` - 'top' | 'bottom' | 'left' | 'right'
- `.compact(boolean)` - Compact mode
- `.siblingsMargin(func)` - Horizontal spacing
- `.childrenMargin(func)` - Vertical spacing
- `.neightbourMargin(func)` - Subtree spacing
- `.compactMarginPair(func)` - Compact pair spacing
- `.compactMarginBetween(func)` - Compact row spacing

**4. Visual Styling**
- `.nodeContent(func)` - HTML generator for nodes
- `.buttonContent(func)` - HTML for expand/collapse button
- `.backgroundColor(color)` - Chart background
- `.nodeDefaultBackground(color)` - Node background
- `.linkColor(color)` - Link stroke color
- `.highlightedLinkColor(color)` - Highlighted link color

**5. Behavior**
- `.duration(ms)` - Transition duration
- `.setActiveNodeCentered(boolean)` - Auto-center on action
- `.expandLevel(number)` - Initial expansion depth
- `.scaleExtent([min, max])` - Zoom range
- `.onNodeClick(callback)` - Click handler

**6. Custom Hooks**
- `.linkUpdate(func)` - Style links after render
- `.nodeUpdate(func)` - Style nodes after render
- `.connectionsUpdate(func)` - Style connections after render

**7. Actions**
- `.render()` - Initial render
- `.fit()` - Zoom to fit all nodes
- `.zoomIn()` - Zoom in
- `.zoomOut()` - Zoom out
- `.fullscreen(selector)` - Enter fullscreen

**8. Node Manipulation**
- `.addNode(obj)` - Add single node
- `.addNodes(array)` - Add multiple nodes
- `.removeNode(id)` - Remove node and descendants
- `.closeNonReportingLineNodes(node)` - Collapse siblings

**9. Node State**
- `.setExpanded(id, flag)` - Expand/collapse node
- `.setCentered(id)` - Center on node
- `.setHighlighted(id)` - Highlight node
- `.setUpToTheRootHighlighted(id)` - Highlight path to root
- `.clearHighlighting()` - Remove all highlights
- `.expandAll()` - Expand all nodes
- `.collapseAll()` - Collapse all nodes

**10. Export**
- `.exportImg(options)` - Export as PNG
- `.exportSvg()` - Export as SVG

---

## 5. Rendering Pipeline

### 5.1 Initial Render Flow

```
┌─────────────────────────────────────────────────────────┐
│ chart.render()                                           │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 1. Validate Data                                         │
│    - Check attrs.data exists and not empty               │
│    - Track all node IDs in addedNodeIds Set              │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Setup Container & SVG                                 │
│    - Get container dimensions                            │
│    - Create/select <svg> element                         │
│    - Set width, height, font-family                      │
│    - Apply background color                              │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Initialize Behaviors (first draw only)                │
│    - Create d3.zoom() behavior                           │
│    - Bind to SVG                                         │
│    - Disable double-click zoom                           │
│    - Set cursor to 'move'                                │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 4. Create Container Groups                               │
│    - <g class="chart">                                   │
│    - <g class="center-group"> (zoom target)              │
│    - <g class="links-wrapper">                           │
│    - <g class="nodes-wrapper">                           │
│    - <g class="connections-wrapper">                     │
│    - <g class="defs-wrapper">                            │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 5. Process Data → Hierarchy                              │
│    - setLayouts({ expandNodesFirst: false })             │
│      └─ d3.stratify() flat data                          │
│      └─ Calculate node dimensions                        │
│      └─ Set expansion states                             │
│      └─ Count direct/total subordinates                  │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 6. Calculate Layout                                      │
│    - Initialize d3-flextree with nodeSize function       │
│    - If compact mode:                                    │
│      └─ calculateCompactFlexDimensions()                 │
│    - Run layout algorithm                                │
│    - If compact mode:                                    │
│      └─ calculateCompactFlexPositions()                  │
│    - Apply layout swap (for orientation)                 │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 7. Render SVG Elements                                   │
│    - update(attrs.root)                                  │
│      ├─ Render Links (paths)                             │
│      ├─ Render Nodes (rect + foreignObject + button)     │
│      └─ Render Custom Connections                        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 8. Apply Initial Transform                               │
│    - Position centerG based on layout orientation        │
│    - Apply lastTransform zoom/pan state                  │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 9. Setup Event Listeners                                 │
│    - Window resize → update SVG width                    │
│    - Set firstDraw = false                               │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
                   Done
```

### 5.2 Update Flow (Subsequent Renders)

```
User Action
    │
    ├─► addNode(obj)
    ├─► removeNode(id)
    ├─► setExpanded(id, flag)
    ├─► onButtonClick (expand/collapse)
    └─► setCentered(id)
    │
    ▼
State Mutation
    │
    └─► Modify attrs.data or node flags
    │
    ▼
updateNodesState() or update(node)
    │
    ├─► Re-stratify data
    ├─► Re-calculate layout
    └─► Re-run compact calculations
    │
    ▼
D3 Data Join (Enter-Update-Exit)
    │
    ├─ ENTER (new elements)
    │   ├─ Create <path> or <g> elements
    │   ├─ Set initial position (parent's position)
    │   └─ Transition to target position
    │
    ├─ UPDATE (existing elements)
    │   ├─ Keep existing elements
    │   ├─ Update data binding
    │   ├─ Transition to new positions
    │   └─ Update styles/attributes
    │
    └─ EXIT (removed elements)
        ├─ Transition to parent position
        ├─ Fade out (opacity: 0)
        └─ Remove from DOM
    │
    ▼
Custom Styling Hooks
    │
    ├─► linkUpdate() - User-defined link styling
    ├─► nodeUpdate() - User-defined node styling
    └─► connectionsUpdate() - User-defined connection styling
    │
    ▼
Auto-centering (if ._centered flag)
    │
    └─► fit({ nodes: [centeredNode], animate: true })
```

### 5.3 SVG Structure

```xml
<svg class="svg-chart-container" width="800" height="600">

  <!-- Definitions (markers, gradients, etc.) -->
  <g class="defs-wrapper">
    <defs>
      <marker id="from_to" ...>...</marker>
      <!-- Dynamic markers for connections -->
    </defs>
  </g>

  <!-- Main chart group (zoom target) -->
  <g class="chart" transform="matrix(...)">

    <!-- Center group (positioned by layout) -->
    <g class="center-group" transform="translate(...)">

      <!-- Links (parent-child connections) -->
      <g class="links-wrapper">
        <path class="link" d="M..." stroke="..." />
        <path class="link" d="M..." stroke="..." />
        ...
      </g>

      <!-- Nodes -->
      <g class="nodes-wrapper">

        <g class="node" transform="translate(...)">
          <!-- Background rectangle -->
          <rect class="node-rect" width="220" height="85" rx="3" />

          <!-- Custom HTML content -->
          <foreignObject class="node-foreign-object" width="220" height="85">
            <div class="node-foreign-object-div">
              <!-- User-defined nodeContent HTML -->
              <div style="...">
                <img src="..." />
                <div>John Doe</div>
                <div>CEO</div>
              </div>
            </div>
          </foreignObject>

          <!-- Expand/collapse button -->
          <g class="node-button-g" transform="translate(...)">
            <rect class="node-button-rect" opacity="0" />
            <foreignObject class="node-button-foreign-object">
              <div class="node-button-div">
                <!-- User-defined buttonContent HTML -->
                <div>▼ 3</div>
              </div>
            </foreignObject>
          </g>
        </g>

        <!-- More nodes... -->
      </g>

      <!-- Custom connections (non-hierarchical) -->
      <g class="connections-wrapper">
        <path class="connection" d="M..." stroke="..." marker-start="..." marker-end="..." />
        ...
      </g>

    </g>
  </g>
</svg>
```

---

## 6. State Management

### 6.1 State Sources

The library manages state in multiple locations:

1. **attrs object** - Configuration and runtime state
2. **Node data objects** - Original user data + internal flags
3. **D3 hierarchy nodes** - Tree structure + positions
4. **DOM state** - Current visual representation

### 6.2 State Flags

**On original data objects** (`node.data`):
- `_expanded: boolean` - Should this node be expanded?
- `_highlighted: boolean` - Is this node highlighted?
- `_upToTheRootHighlighted: boolean` - Highlight path to root?
- `_centered: boolean` - Should center on this node?
- `_centeredWithDescendants: boolean` - Include descendants in centering?
- `_directSubordinates: number` - Count of direct children
- `_totalSubordinates: number` - Count of all descendants
- `_filtered: boolean` - Mark for removal
- `_filteredOut: boolean` - Filtered from view

**On D3 hierarchy nodes**:
- `children: Array` - Expanded children (visible)
- `_children: Array` - Collapsed children (hidden)
- `x: number` - Calculated X position
- `y: number` - Calculated Y position
- `x0: number` - Previous X (for transitions)
- `y0: number` - Previous Y (for transitions)
- `width: number` - Node width
- `height: number` - Node height
- `parent: Node` - Parent node reference
- `depth: number` - Level in tree (0 = root)
- `flexCompactDim: [number, number]` - Compact layout dimensions
- `compactEven: boolean` - Even position in compact layout
- `firstCompact: boolean` - First in compact group
- `firstCompactNode: Node` - Reference to first compact sibling
- `row: number` - Row in compact layout

### 6.3 State Synchronization

```
User Data          attrs.data (flat array)
    │                      │
    │  d3.stratify()       │
    │                      │
    ▼                      ▼
Hierarchy          attrs.root (tree)
    │                      │
    │  Layout calc         │
    │                      │
    ▼                      ▼
Positions          node.x, node.y
    │                      │
    │  Render              │
    │                      │
    ▼                      ▼
DOM                SVG elements
```

**Synchronization Points**:
1. Data mutation → `updateNodesState()` → Re-stratify → Re-render
2. Expansion toggle → Swap `children` ↔ `_children` → `update()`
3. Visual change → Direct DOM manipulation (via hooks)

### 6.4 Expansion State Algorithm

```javascript
// Collapse all, then selectively expand
function setLayouts({ expandNodesFirst }) {
  // 1. Stratify data
  attrs.root = d3.stratify()(...);

  // 2. Optionally expand all first
  if (expandNodesFirst && attrs.root.children) {
    attrs.root.children.forEach(expand);
  }

  // 3. Collapse all
  if (attrs.root.children) {
    attrs.root.children.forEach(collapse);
  }

  // 4. Collapse root if expandLevel === 0
  if (attrs.expandLevel === 0) {
    attrs.root._children = attrs.root.children;
    attrs.root.children = null;
  }

  // 5. Expand only nodes with _expanded flag
  [attrs.root].forEach(expandSomeNodes);
}

function expandSomeNodes(node) {
  if (node.data._expanded) {
    // Walk up and expand all ancestors
    let parent = node.parent;
    while (parent) {
      if (parent._children) {
        parent.children = parent._children;
      }
      parent = parent.parent;
    }
  }
  // Recurse on children
  if (node.children) {
    node.children.forEach(expandSomeNodes);
  }
  if (node._children) {
    node._children.forEach(expandSomeNodes);
  }
}
```

---

## 7. Key Algorithms

### 7.1 Compact Layout Algorithm

**Purpose**: Arrange leaf nodes in a grid to save space

**Location**: `calculateCompactFlexDimensions()` + `calculateCompactFlexPositions()`

**Steps**:

1. **Identify Compact Candidates**
   ```javascript
   // For each parent with 2+ leaf children
   const compactChildren = node.children.filter(d => !d.children);
   if (compactChildren.length < 2) return;
   ```

2. **Assign Rows & Columns**
   ```javascript
   compactChildren.forEach((child, i) => {
     child.row = Math.floor(i / 2);  // 2 nodes per row
     child.compactEven = (i % 2 === 0);
   });
   ```

3. **Calculate Dimensions**
   ```javascript
   // Max width of even/odd columns
   const evenMax = max(compactChildren.filter(d => d.compactEven), sizeColumn);
   const oddMax = max(compactChildren.filter(d => !d.compactEven), sizeColumn);
   const columnSize = max(evenMax, oddMax) * 2;

   // Sum of row heights
   const rowSize = sum(groupedByRow, row => max(row, sizeRow));
   ```

4. **Position Nodes**
   ```javascript
   // First node gets full dimensions
   firstNode.flexCompactDim = [columnSize, rowSize];

   // Others get [0, 0] to not affect layout
   others.forEach(node => {
     node.flexCompactDim = [0, 0];
   });

   // Later, position based on row/column
   node.x = firstNode.x + (isEven ? 0.25 : 0.75) * columnSize;
   node.y = firstNode.y + cumulativeRowHeight[node.row];
   ```

### 7.2 Link Path Generation

**Diagonal (Vertical Layouts)**:
```javascript
diagonal(source, target, mid) {
  const x = source.x;
  const y = source.y;
  const ex = target.x;
  const ey = target.y;

  // Curve radius
  const r = min(35, abs(ex - x) / 2, abs(ey - y) / 2);

  // Distances
  const h = abs(ey - y) / 2 - r;  // Vertical straight segment
  const w = abs(ex - x) - r * 2;   // Horizontal straight segment

  // Build path with rounded corners
  return `
    M ${x} ${y}
    L ${x} ${y + h}
    Q ${x} ${y + h + r} ${x + r} ${y + h + r}
    L ${ex - r} ${y + h + r}
    Q ${ex} ${y + h + r} ${ex} ${ey - h}
    L ${ex} ${ey}
  `;
}
```

**hdiagonal (Horizontal Layouts)**:
Similar logic but swaps X/Y axes.

---

## 8. Performance Considerations

### 8.1 Bottlenecks

1. **Large Datasets** (>500 nodes)
   - All nodes rendered to DOM (no virtualization)
   - Layout calculation is O(n)
   - SVG performance degrades

2. **Frequent Updates**
   - Full re-stratification on data change
   - Layout recalculation
   - All nodes re-rendered (even if positions unchanged)

3. **Compact Mode**
   - Additional passes for dimension/position calculation
   - More complex logic

### 8.2 Optimizations Present

1. **Enter-Update-Exit Pattern**
   - Only animates changed nodes
   - Reuses existing DOM elements

2. **Flextree Algorithm**
   - More efficient than standard D3 tree layout
   - Better space utilization

3. **Canvas Text Measurement**
   - Uses 2D canvas context for text width
   - Faster than SVG measurement

### 8.3 Potential Improvements

1. **Virtual Scrolling** - Only render visible nodes
2. **Debouncing** - Limit update frequency on rapid changes
3. **Web Workers** - Offload layout calculation
4. **Memoization** - Cache layout results for unchanged subtrees
5. **Level-of-Detail** - Simplify far-away nodes

---

## 9. Integration Patterns

### 9.1 Vanilla JavaScript

```javascript
const chart = new d3.OrgChart()
  .container('#org-chart')
  .data(employees)
  .render();

// Dynamic updates
document.getElementById('add-btn').addEventListener('click', () => {
  chart.addNode({
    id: 'new-' + Date.now(),
    parentId: 'CEO',
    name: 'New Hire'
  });
});
```

### 9.2 React Integration

```jsx
import { useEffect, useRef } from 'react';
import { OrgChart } from 'd3-org-chart';

function OrgChartComponent({ data }) {
  const chartRef = useRef(null);
  const containerRef = useRef(null);

  useEffect(() => {
    if (!chartRef.current) {
      chartRef.current = new OrgChart()
        .container(containerRef.current)
        .data(data)
        .render();
    } else {
      chartRef.current.data(data).render();
    }
  }, [data]);

  return <div ref={containerRef} />;
}
```

### 9.3 Vue Integration

```vue
<template>
  <div ref="chartContainer"></div>
</template>

<script>
import { OrgChart } from 'd3-org-chart';

export default {
  props: ['employees'],
  data() {
    return { chart: null };
  },
  mounted() {
    this.chart = new OrgChart()
      .container(this.$refs.chartContainer)
      .data(this.employees)
      .render();
  },
  watch: {
    employees(newData) {
      this.chart.data(newData).render();
    }
  }
};
</script>
```

---

## 10. Browser Compatibility

### Requirements

- **SVG Support**: Required (all modern browsers)
- **ES6 Features**: Classes, arrow functions, destructuring
  - Requires transpilation for IE11
- **ForeignObject**: For custom HTML in nodes
  - Not supported in IE11 (falls back to SVG text)
- **Canvas 2D Context**: For text measurement

### Tested Browsers

- Chrome/Edge 90+
- Firefox 88+
- Safari 14+

---

**End of Architecture Documentation**
