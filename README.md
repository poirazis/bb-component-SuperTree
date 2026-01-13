# Super Tree Component for Budibase

A powerful and flexible Tree View/List component for Budibase applications that supports hierarchical data display with advanced features including selection, menus, search, and multiple visualization modes.

## 🚀 Features

- **Multiple Tree Types**: Flat, Recursive, and Group By structures
- **Flexible Display Modes**: Tree View and List View options
- **Advanced Selection**: Single/multi-selection with checkboxes
- **Context Menus**: Configurable menus for nodes, groups, and headers
- **Built-in Search**: Client-side and server-side search capabilities
- **Styling Control**: Extensive customization options for colors, icons, and layout
- **Data Integration**: Full support for Budibase data sources with filtering and sorting
- **Nested Trees**: Support for nested tree structures within tree nodes
- **Event System**: Rich event handling for node interactions
- **Responsive Design**: Adaptive sizing and flex mode support

## 📦 Installation

This is a custom Budibase component plugin that needs to be imported into your Budibase instance.

### Local Development

Build the plugin package:

```bash
npm run build
```

For development with auto-rebuild:

```bash
npm run watch
```

### Import to Budibase

After building, you have several options to import the plugin:

1. **Via File Upload**: Upload the generated plugin tarball
2. **Via NPM**: Publish and install via NPM package URL
3. **Via GitHub**: Host on GitHub with release tarballs
4. **Via URL**: Direct tarball URL hosting

#### In Budibase Portal:

1. Go to **Plugins** section
2. Click **"Add Plugin"**
3. Choose your preferred import method (File Upload, NPM, GitHub, or URL)
4. Once imported, the component will be available in the **Plugins** section of the component library

#### Hot Reloading (Development)

For faster development with hot reloading:

1. **Set up Budibase CLI**: Install and initialize Budibase CLI
2. **Configure plugin directory**:
   ```bash
   budi hosting --watch-plugin-dir /path-to-your-plugins-directory
   ```
3. **Start Budibase hosting**:
   ```bash
   budi hosting --start
   ```
4. **Run plugin watch** in your component directory:
   ```bash
   npm run watch
   ```

### Using the Component

Once imported:

1. Open your Budibase app in the builder
2. Find **SuperTree** in the component library under **Plugins** section
3. Drag and drop it into your design
4. Configure properties through the builder UI

## 🎯 Core Concepts

### Tree Types

- **Flat**: Simple tree structure without hierarchy
- **Recursive**: Parent-child relationships based on a join field
- **Group By**: Data grouped by one or more fields with expandable groups

### Control Types

- **Tree View**: Hierarchical tree structure with expand/collapse functionality
- **List View**: Flat list layout with grouped sections

### Selection Modes

- Single selection (default)
- Multi-selection with configurable limits
- Checkbox-based selection
- Programmatic pre-selection

## ⚙️ Configuration Properties

### Data Source Settings

| Property         | Type       | Description                     | Default                |
| ---------------- | ---------- | ------------------------------- | ---------------------- |
| `datasource`     | DataSource | Budibase data source            | -                      |
| `idColumn`       | String     | Column used for node IDs        | `id`                   |
| `labelColumn`    | String     | Column used for node labels     | Primary display column |
| `labelTemplate`  | String     | Template string for node labels | -                      |
| `structuredData` | Boolean    | Use structured data format      | `false`                |

### Tree Configuration

| Property      | Type                               | Default | Description                        |
| ------------- | ---------------------------------- | ------- | ---------------------------------- |
| `treeType`    | `flat` \| `recursive` \| `groupBy` | `flat`  | Tree structure type                |
| `controlType` | `tree` \| `list`                   | `tree`  | Display mode                       |
| `joinField`   | String                             | -       | Field for recursive relationships  |
| `groupFields` | Array                              | -       | Fields to group by (Group By mode) |

### Display Options

| Property          | Type              | Default | Description              |
| ----------------- | ----------------- | ------- | ------------------------ |
| `rootless`        | Boolean           | `true`  | Hide root node container |
| `header`          | Boolean           | `true`  | Show tree header         |
| `branchName`      | String            | -       | Custom root node text    |
| `collapsed`       | Boolean           | `true`  | Start tree collapsed     |
| `showCount`       | Boolean           | `true`  | Show item counts         |
| `chevronPosition` | `left` \| `right` | `left`  | Chevron icon position    |

### Search & Filter

| Property     | Type                            | Default  | Description            |
| ------------ | ------------------------------- | -------- | ---------------------- |
| `searchMode` | `false` \| `client` \| `server` | `client` | Search implementation  |
| `filter`     | Array                           | -        | Pre-configured filters |
| `sortColumn` | String                          | -        | Sort field             |
| `sortOrder`  | `ascending` \| `descending`     | -        | Sort direction         |
| `limit`      | Number                          | `50`     | Row limit              |

### Selection & Interaction

| Property           | Type    | Default | Description               |
| ------------------ | ------- | ------- | ------------------------- |
| `nodeSelection`    | Boolean | `false` | Enable node selection     |
| `maxNodeSelection` | Number  | `1`     | Maximum selections        |
| `checkboxes`       | Boolean | -       | Show selection checkboxes |
| `selectedIds`      | String  | -       | Pre-selected node IDs     |

### Styling & Appearance

| Property           | Type    | Description            | Scope  |
| ------------------ | ------- | ---------------------- | ------ |
| `quiet`            | Boolean | Minimal styling        | All    |
| `nodeIcon`         | String  | Node icon              | Nodes  |
| `nodeIconTemplate` | String  | Dynamic icon template  | Nodes  |
| `nodeFGColor`      | String  | Foreground color       | Nodes  |
| `nodeBGColor`      | String  | Background color       | Nodes  |
| `nodeIconColor`    | String  | Icon color             | Nodes  |
| `groupNodeIcon`    | String  | Group node icon        | Groups |
| `groupFGColor`     | String  | Group foreground color | Groups |
| `groupBGColor`     | String  | Group background color | Groups |

### Menus & Actions

| Property          | Type    | Default | Description         |
| ----------------- | ------- | ------- | ------------------- |
| `headerMenu`      | Boolean | `false` | Enable header menu  |
| `headerMenuItems` | Array   | -       | Header menu actions |
| `nodeMenuItems`   | Array   | -       | Node menu actions   |
| `groupMenuItems`  | Array   | -       | Group menu actions  |

### Events

| Event           | Context                     | Description            |
| --------------- | --------------------------- | ---------------------- |
| `onNodeSelect`  | Node data, selection state  | Node selection change  |
| `onNodeClick`   | Node data, click event      | Node clicked           |
| `onGroupSelect` | Group data, selection state | Group selection change |
| `onGroupClick`  | Group data, click event     | Group clicked          |

## 📋 Usage Examples

### Basic Tree View

In Budibase builder, configure these settings:

- **Data Source**: Select your data table/query
- **Control Type**: Tree View
- **Tree Type**: Flat
- **Selection**: Enable node selection
- **Search**: Client-side

### Recursive Hierarchical Tree

Configure for parent-child relationships:

- **Data Source**: Your hierarchical data source
- **Tree Type**: Recursive
- **Join Field**: Specify the parent ID field
- **Rootless**: False (to show root container)
- **Branch Name**: Set custom root label
- **Node Menu**: Enable and configure actions

### Grouped Data Tree

For data grouped by categories:

- **Data Source**: Your grouped data source
- **Tree Type**: Group By
- **Group Fields**: Select fields to group by
- **Group Selection**: Enable if groups should be selectable
- **Max Node Selection**: Set selection limits as needed

### List View with Multiple Selection

Configure for flat list layout:

- **Control Type**: List View
- **Tree Type**: Flat (or Group By)
- **Selection**: Enable with checkboxes
- **Max Selection**: Configure number of items users can select

## 🎨 Styling & Theming

### Dynamic Node Styling

Use template strings to customize node appearance based on data:

```json
{
  "nodeIconTemplate": "{{#if completed}}✓{{/if}}",
  "nodeFGColor": "{{#if priority === 'high'}}#e74c3c{{/if}}",
  "nodeBGColor": "{{#if selected}}#e8f5e8{{/if}}"
}
```

### Group Styling

```json
{
  "groupNodeIcon": "folder-fill",
  "groupFGColor": "#2c3e50",
  "groupBGColor": "#ecf0f1"
}
```

## 🔧 Advanced Features

### Context-Aware Features

- **Global Context**: Selected items and IDs available in `{{ SuperTree.selected }}`
- **Local Context**: Access group values, node children, and current data
- **Parent Tree Context**: Nested trees inherit options from parent components

### Search Functionality

- **Client-side**: Fast local filtering with highlighting
- **Server-side**: Database-level search with reduced data transfer
- **Disabled**: No search functionality

### Menu System

Configure contextual actions for different tree elements:

```json
{
  "nodeMenuItems": [
    {
      "text": "Edit Item",
      "icon": "edit",
      "onClick": {
        "action": "updateRow",
        "table": "{{ yourTable }}",
        "rowId": "{{ row.id }}"
      }
    }
  ]
}
```

### Nested Trees

The SuperTree component supports complex nested structures:

- **Parent Tree Context**: Configure nested trees to inherit options from parent components
- **Custom Content**: Add custom content to tree nodes through the Budibase builder
- **Data Context**: Access parent node information when configuring nested tree components

## 🤝 Integration

### Budibase Data Sources

- **Tables**: Direct table binding with column mapping
- **Views**: Filtered and transformed data
- **External APIs**: REST API integration
- **Custom Queries**: Advanced filtering and joins

### Events & Actions

- **Trigger Automations**: Execute server-side logic on user interactions
- **Update Components**: Sync state with other UI components
- **Navigate**: Trigger routing changes
- **API Calls**: Call external services

### Context Sharing

Access SuperTree data in other components using Budibase bindings:

- **Selected Items**: `{{ SuperTree.selected }}`
- **Selected IDs**: `{{ SuperTree.selectedIds }}`
- **Selected Path**: `{{ SuperTree.selectedPath }}`
- **Current Group**: `{{ SuperTree.group }}`

Use these bindings in other components to access the tree's current state and selections.

## 🐛 Troubleshooting

### Common Issues

- **Data not displaying**: Check datasource configuration and column mappings
- **Search not working**: Verify search mode setting and label column configuration
- **Selection not saving**: Ensure node IDs match data source primary keys
- **Styling not applied**: Confirm template syntax for dynamic styling

### Performance Considerations

- Use server-side search for large datasets (>1000 items)
- Implement pagination for very large tree structures
- Consider flat tree type for simple lists without hierarchy
- Use checkboxes sparingly with multi-selection to avoid UI clutter

## 🤝 Contributing

Contributions welcome! Please ensure code follows Budibase component standards and includes proper TypeScript definitions.

## 📄 License

This component is part of the Budibase ecosystem. See [Budibase Licensing](https://budibase.com/open-source/) for details.
