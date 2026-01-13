<script>
  import TreeHeader from "../lib/TreeHeader.svelte";
  import { getContext, setContext, tick } from "svelte";
  import { writable } from "svelte/store";

  import Tree from "../lib/Tree.svelte";
  const {
    styleable,
    ActionTypes,
    Provider,
    fetchData,
    processStringSync,
    QueryUtils,
    memo,
    API,
    builderStore,
    screenStore,
    notificationStore,
    enrichButtonActions,
  } = getContext("sdk");

  const component = getContext("component");
  const parentTree = getContext("superTreeOptions");
  const parentFilter = getContext("superTreeFilter");
  const allContext = getContext("context");

  // Helper functions
  const brain = {
    lookupComponent: (components, id) => {
      let parent;
      let pos = components?.findIndex((comp) => comp._id == id);
      if (pos > -1) return components[pos];
      else
        components?.forEach((comp) => {
          if (!parent) parent = brain.lookupComponent(comp._children, id);
        });

      return parent;
    },

    getNode: (nodes, id) => {
      if (!nodes || !Array.isArray(nodes)) return null;
      for (let node of nodes) {
        if (node.id === id) return node;
        if (node.children) {
          let found = brain.getNode(node.children, id);
          if (found) return found;
        }
      }
      return null;
    },

    getDescendants: (node) => {
      let ids = [];
      if (node.children && Array.isArray(node.children)) {
        for (let child of node.children) {
          ids.push(child.id);
          ids.push(...brain.getDescendants(child));
        }
      }
      return ids;
    },

    getAncestors: (tree, nodeId) => {
      4;
      function findAncestors(node, targetId, ancestors = []) {
        if (node.id === targetId) {
          return ancestors;
        }

        if (!node.children || !Array.isArray(node.children)) return null;

        for (const child of node.children) {
          const result = findAncestors(child, targetId, [
            ...ancestors,
            node.id,
          ]);
          if (result) {
            return result;
          }
        }

        return null;
      }

      if (!tree || !Array.isArray(tree)) return null;

      for (const rootNode of tree) {
        if (rootNode.id === nodeId) {
          return [];
        }

        const ancestors = findAncestors(rootNode, nodeId);
        if (ancestors) {
          return ancestors;
        }
      }

      return null;
    },

    getBreadcrumbs: (selectedAncestors, selectedNodes, rootNodes) => {
      if (selectedNodes.length > 1) return null;

      let ancestorLabels = selectedAncestors
        .map((id) => {
          let node = brain.getNode(rootNodes, id);
          return node ? node.label : null;
        })
        .filter((x) => x != null);

      if (selectedNodes.length > 0) {
        let selectedNode = brain.getNode(rootNodes, selectedNodes[0]);
        if (selectedNode) {
          ancestorLabels.push(selectedNode.label);
        }
      }

      return ancestorLabels.length > 0
        ? "/ " + ancestorLabels.join(" / ")
        : "/";
    },

    handleNodeSelect: (e) => {
      if (inBuilder) return;

      if (e.detail.isGroup) {
        if ($selectedGroups.includes(e.detail.id)) {
          selectedGroups.set([]);
        } else {
          selectedGroups.set([e.detail.id]);
        }

        if ($selectedGroups.length) onGroupSelect?.({ group: e.detail.id });

        selectedNodes.set([]);
        selectedRows.set([]);
        selectedAncestors.set([]);
        selectedDescendants.set([]);

        return;
      }

      let row = e.detail.row;

      let index = $selectedNodes.indexOf(e.detail.id);
      if (index > -1) {
        // deselect
        $selectedNodes.splice(index, 1);
        $selectedNodes = $selectedNodes;
      } else {
        // select
        selectedGroups.set([]);
        let max = maxNodeSelection || 1;
        if (max == 0 || $selectedNodes.length < max) {
          $selectedNodes = [...$selectedNodes, e.detail.id];
        } else if (max == 1) {
          $selectedNodes = [e.detail.id];
        } else {
          notificationStore.actions.warning(
            "Cannot select more than " + max + " items"
          );
          return;
        }
      }

      if ($selectedNodes.length == 1) {
        let node = brain.getNode(rootNodes, $selectedNodes[0]);
        if (node) {
          selectedAncestors.set(
            brain.getAncestors(rootNodes, $selectedNodes[0]) || []
          );
          selectedDescendants.set(brain.getDescendants(node));
        }
      } else {
        selectedAncestors.set([]);
        selectedDescendants.set([]);
      }

      $selectedRows = $fetch?.rows?.filter((row) =>
        $selectedNodes.find((x) => x == row[idColumn])
      );

      if ($selectedRows.length && onNodeSelect) {
        onNodeSelect({ row });
      }
    },

    loadSelections: () => {
      let ids = selectedIds
        ? selectedIds
            .split(",")
            .map((id) => (idColumnType === "number" ? Number(id) : id))
        : [];
      if (ids.length) selectedNodes.set(ids);
    },

    createFetch: (datasource) => {
      return fetchData({
        API,
        datasource,
        options: {
          query,
          sortColumn,
          sortOrder,
          limit,
          paginate,
        },
      });
    },

    extendQuery: (defaultQuery, extensions) => {
      const extensionValues = Object.values(extensions || {});
      let extendedQuery = { ...defaultQuery };
      extensionValues.forEach((extension) => {
        Object.entries(extension || {}).forEach(([operator, fields]) => {
          extendedQuery[operator] = {
            ...extendedQuery[operator],
            ...fields,
          };
        });
      });
      return { ...extendedQuery, onEmptyFilter: "all" };
    },

    enrichNode: (row, idx, options = {}) => {
      const {
        isGroup = false,
        groupValue,
        children = [],
        visible = true,
      } = options;

      // Determine renderSlot based on slotPosition string value
      const shouldRenderSlot = (() => {
        if (!hasSlot) return false;
        switch (slotPosition) {
          case false:
          case "disabled":
            return false;
          case "all":
            return true;
          case "leaf":
            return !children || children.length === 0;
          case "branch":
            return children && children.length > 0;
          case "group":
            return isGroup;
          case "after":
            // Handled elsewhere (e.g., after the tree), so false here
            return false;
          default:
            return false;
        }
      })();

      return {
        id: row[idColumn] || `${groupValue}`,
        renderSlot: shouldRenderSlot,
        isGroup,
        visible,
        quiet: isGroup ? quiet : undefined,
        row: !isGroup ? row : undefined,
        open: $builderStore.inBuilder && $component.children && idx === 0,
        icon: nodeIconTemplate
          ? processStringSync(nodeIconTemplate, { [comp_id]: { row } })
          : nodeIcon,
        iconColor: nodeIconColor
          ? processStringSync(nodeIconColor, { [comp_id]: { row } })
          : undefined,
        label: labelTemplate
          ? processStringSync(labelTemplate, { [comp_id]: { row } })
          : (row[labelColumn || primaryDisplay] ?? "Not Set"),
        color: nodeFGColor
          ? processStringSync(nodeFGColor, { [comp_id]: { row } })
          : undefined,
        bgColor: nodeBGColor
          ? processStringSync(nodeBGColor, { [comp_id]: { row } })
          : undefined,
        children,
        selectable: !isGroup,
      };
    },

    buildTree: (rows) => {
      if (!rows || rows.length === 0) {
        rootNodes = [];
        return;
      }

      selectedNodes.set([]);
      selectedRows.set([]);
      selectedAncestors.set([]);
      selectedDescendants.set([]);
      selectedGroups.set([]);

      groupByValues.clear();
      rootNodes = [];

      if (structuredData) {
        rootNodes = rows;
        return;
      }

      if (treeType === "groupBy" && groupFields?.length > 0) {
        rootNodes = brain.buildGroupNodes(rows, groupFields, 0);
      } else if (recursive) {
        rows
          .filter((x) => !x[joinField])
          .forEach((row, idx) => {
            rootNodes.push({
              ...brain.enrichNode(row, idx, {
                children: brain.getChildNodes(row, rows),
              }),
            });
          });
      } else {
        rows.forEach((row, idx) =>
          rootNodes.push({
            ...brain.enrichNode(row, idx, {
              children: brain.getListColumnChildren(row),
            }),
          })
        );
      }
    },

    buildGroupNodes: (rows, groupFields, level) => {
      const currentField = groupFields[level];
      const nextLevel = level + 1;
      const isLastLevel = nextLevel >= groupFields.length;

      // Handle case where the group field is an array, either a relationshiup or a multi-select field
      const groupValues = new Set(
        rows.map((row) => {
          if (Array.isArray(row[currentField])) {
            return (
              row[currentField][0]?.primaryDisplay ||
              row[currentField][0]?._id ||
              row[currentField][0]
            );
          }
          return row[currentField];
        })
      );

      return Array.from(groupValues).map((value, idx) => {
        const groupRows = rows.filter((row) =>
          brain.isChild(row[currentField], value)
        );

        const children = isLastLevel
          ? brain.getGroupChildNodes(value, groupRows)
          : brain.buildGroupNodes(groupRows, groupFields, nextLevel);

        // Compute label first
        const computedLabel = groupNodeLabel
          ? processStringSync(groupNodeLabel, {
              ...$allContext,
              [comp_id]: {
                ...$allContext[comp_id],
                group: value,
              },
            })
          : Array.isArray(value)
            ? value.join(", ")
            : value || "Not Set";

        // Compute renderSlot dynamically (same logic as enrichNode)
        const shouldRenderSlot = (() => {
          if (!hasSlot) return false;
          switch (slotPosition) {
            case false:
            case "disabled":
              return false;
            case "all":
              return true;
            case "leaf":
              return !children || children.length === 0;
            case "branch":
              return children && children.length > 0;
            case "group":
              return true; // Since this is a groupBranch
            case "after":
              return false;
            default:
              return false;
          }
        })();

        return {
          selectable: groupSelectable,
          renderSlot: shouldRenderSlot, // Now dynamic, based on slotPosition
          id: computedLabel,
          visible: true,
          open: $builderStore.inBuilder && $component.children && idx === 0,
          icon: groupNodeIcon,
          label: computedLabel,
          isGroup: true,
          color: groupFGColor
            ? processStringSync(groupFGColor, {
                ...$allContext,
                [comp_id]: {
                  ...$allContext[comp_id],
                  group: value,
                },
              })
            : undefined,
          bgColor: groupBGColor
            ? processStringSync(groupBGColor, {
                ...$allContext,
                [comp_id]: {
                  ...$allContext[comp_id],
                  group: value,
                },
              })
            : undefined,
          children,
        };
      });
    },

    getGroupChildNodes: (groupValue, groupRows) => {
      return groupRows.map((row, idx) =>
        brain.enrichNode(row, idx, {
          groupValue,
          children: [...brain.getChildNodes(row)],
        })
      );
    },

    getChildNodes: (row, rows) => {
      let children = [];

      if (recursive) {
        rows
          .filter((x) => x[joinField] == row[idColumn])
          .forEach((row, idx) => {
            children.push({
              ...brain.enrichNode(row, idx, {
                children: brain.getChildNodes(row, rows),
                visible: true,
              }),
              quiet,
            });
          });
      }
      return [...children, ...brain.getListColumnChildren(row)];
    },

    getListColumnChildren: (row) => {
      // Unique listColumns by name to prevent duplicates
      const uniqueCols = listColumns.filter(
        (col, index, self) =>
          index === self.findIndex((c) => c.name === col.name)
      );

      if (uniqueCols.length === 1) {
        // If only one list column, add items as direct descendants
        const col = uniqueCols[0];
        const linked = row[col.name];
        if (!Array.isArray(linked) || linked.length === 0) return [];

        return linked.map((item) => ({
          id: item._id || item.id || `item_${Math.random()}`,
          label: item.primaryDisplay || item.label || item.name || "Unknown",
          isGroup: false,
          row: item,
          visible: true,
          open: false,
          icon: nodeIcon,
          iconColor: nodeIconColor,
          color: nodeFGColor,
          bgColor: nodeBGColor,
          renderSlot: false,
          selectable: true,
        }));
      } else {
        // If more than one list column, create branches
        return uniqueCols
          .map((col) => {
            const linked = row[col.name];
            if (!Array.isArray(linked) || linked.length === 0) return null;

            const groupChildren = linked.map((item) => ({
              id: item._id || item.id || `item_${Math.random()}`,
              label:
                item.primaryDisplay || item.label || item.name || "Unknown",
              isGroup: false,
              row: item,
              visible: true,
              open: false,
              icon: nodeIcon,
              iconColor: nodeIconColor,
              color: nodeFGColor,
              bgColor: nodeBGColor,
              renderSlot: false,
              selectable: true,
            }));

            return {
              id: `${row[idColumn]}_${col.id}_group`,
              label: col.displayName,
              isGroup: true,
              children: groupChildren,
              visible: true,
              open: false,
              icon: nodeIcon,
              iconColor: nodeIconColor,
              color: nodeFGColor,
              bgColor: nodeBGColor,
              renderSlot: false,
              selectable: false,
              quiet: quiet,
            };
          })
          .filter((x) => x);
      }
    },

    getAggregatedListColumnChildren: (groupRows) => {
      // Unique listColumns by name to prevent duplicates
      const uniqueCols = listColumns.filter(
        (col, index, self) =>
          index === self.findIndex((c) => c.name === col.name)
      );

      if (uniqueCols.length === 1) {
        // If only one list column, add items as direct descendants
        const col = uniqueCols[0];
        // Collect all linked items from all rows
        const allLinked = groupRows
          .flatMap((row) => row[col.name] || [])
          .filter(
            (item, index, arr) =>
              arr.findIndex((i) => i._id === item._id) === index
          ); // unique by _id

        if (allLinked.length === 0) return [];

        return allLinked.map((item) => ({
          id: item._id || item.id || `item_${Math.random()}`,
          label: item.primaryDisplay || item.label || item.name || "Unknown",
          isGroup: false,
          row: item,
          visible: true,
          open: false,
          icon: nodeIcon,
          iconColor: nodeIconColor,
          color: nodeFGColor,
          bgColor: nodeBGColor,
          renderSlot: false,
          selectable: true,
        }));
      } else {
        // If more than one list column, create branches
        return uniqueCols
          .map((col) => {
            // Collect all linked items from all rows
            const allLinked = groupRows
              .flatMap((row) => row[col.name] || [])
              .filter(
                (item, index, arr) =>
                  arr.findIndex((i) => i._id === item._id) === index
              ); // unique by _id

            if (allLinked.length === 0) return null;

            const groupChildren = allLinked.map((item) => ({
              id: item._id || item.id || `item_${Math.random()}`,
              label:
                item.primaryDisplay || item.label || item.name || "Unknown",
              isGroup: false,
              row: item,
              visible: true,
              open: false,
              icon: nodeIcon,
              iconColor: nodeIconColor,
              color: nodeFGColor,
              bgColor: nodeBGColor,
              renderSlot: false,
              selectable: true,
            }));

            return {
              id: `aggregated_${col.id}_group`,
              label: col.displayName,
              isGroup: true,
              children: groupChildren,
              visible: true,
              open: false,
              icon: nodeIcon,
              iconColor: nodeIconColor,
              color: nodeFGColor,
              bgColor: nodeBGColor,
              renderSlot: false,
              selectable: false,
              quiet: quiet,
            };
          })
          .filter((x) => x);
      }
    },

    isChild: (row, group) => {
      // Check if both are identical
      if (row == group) return true;

      // Check if both are arrays
      if (Array.isArray(row)) {
        return (
          row[0]?.primaryDisplay == group ||
          row[0]?._id == group ||
          row[0] == group
        );
      }
    },

    handleNodeAction: async (e) => {
      let cmd = enrichButtonActions(e.detail.onClick, {
        ...$allContext,
        [comp_id]: {
          ...context,
          ...e.detail,
          row: e.detail.row,
          group: e.detail.group,
        },
      });

      await cmd?.();
    },

    handleSearch: (e) => {
      if ($filtering) return;

      $treeFilter = e.detail;
      $selectedNodes = [];
      $selectedRows = [];
      selectedAncestors.set([]);
      selectedDescendants.set([]);

      if (searchMode == "client") {
        if (e.detail) {
          clearFilter(rootNodes);
          filterTree(rootNodes, e.detail);
        } else clearFilter(rootNodes);
        return;
      } else if (searchMode) {
        if (e.detail) {
          searchFilter = [
            {
              field: labelColumn || primaryDisplay,
              operator: "fuzzy",
              value: e.detail,
              type: "string",
              valueType: "Value",
            },
          ];
        } else {
          searchFilter = [];
        }
      }
    },

    getPrimaryDisplay: (definition) => {
      if (definition) {
        if (definition.primaryDisplay) return definition.primaryDisplay;
        else return Object.keys(definition.schema)[0];
      }
    },
  };

  const parent = brain.lookupComponent(
    $screenStore.activeScreen.props._children,
    $component.path.at(-2)
  );

  const nested = parent?._component == "plugin/bb-component-SuperTree";

  export let rootNodeName;
  export let controlType = "tree";
  export let treeType = "flat";
  export let chevronPosition = "left";
  export let collapsed;
  export let quiet;

  export let searchMode;
  export let disabled;
  export let rootless;
  export let rootButtons;

  export let header;
  export let headerText;
  export let headerMenu;
  export let headerMenuItems;
  export let headerShowButtons;
  export let headerMenuIcon = "ri-more-2-fill";

  export let nodeIcon;
  export let nodeIconTemplate;
  export let nodeMenuIcon = "ri-more-fill";
  export let nodeMenuItems = [];
  export let nodeFGColor;
  export let nodeBGColor;
  export let nodeIconColor;

  export let checkboxes;
  export let maxNodeSelection = 1;
  export let selectedIds;

  export let groupNodeIcon;
  export let groupNodeLabel;
  export let groupSelectable;
  export let groupMenuItems = [];
  export let groupFGColor;
  export let groupBGColor;
  export let showCount = true;

  export let datasource;
  export let filter = [];
  export let sortColumn;
  export let sortOrder;
  export let limit;
  export let paginate = true;
  export let labelColumn;
  export let labelTemplate;
  export let structuredData;

  export let listColumns = [];

  export let joinField;
  export let groupFields = [];

  export let flex;
  export let idColumn;

  export let slotPosition = false;

  // Events
  export let onNodeSelect;
  export let onGroupSelect;

  // Use Stores for non-primitive data types
  const dataSourceStore = memo(datasource);
  $: dataSourceStore.set(datasource);

  const treeOptions = memo({});
  const headerMenuItemsStore = memo(headerMenuItems);
  $: headerMenuItemsStore.set(headerMenuItems);

  const nodeMenuItemsStore = memo(nodeMenuItems);
  $: nodeMenuItemsStore.set(nodeMenuItems);

  const groupMenuItemsStore = memo(groupMenuItems);
  $: groupMenuItemsStore.set(groupMenuItems);

  let groupByValues = new Set();
  let rootNodes = [];
  let selectedNodes = new writable([]);
  let selectedAncestors = new writable([]);
  let selectedDescendants = new writable([]);
  let selectedRows = new writable([]);
  let selectedGroups = writable([]);
  let menuStore = new writable(null);
  let query = {};
  let defaultQuery;
  let searchFilter;
  let filtering = writable(false);
  let treeFilter = writable(null);
  let currentRow = writable(null);
  let hasMatches = false;

  $: comp_id = $component.id;
  $: inBuilder = $builderStore.inBuilder;
  $: quiet = $parentTree ? $parentTree.quiet : quiet;
  $: disabled = $parentTree ? $parentTree.disabled : disabled;
  $: searchable = searchMode && !disabled;

  $: headerButtons = headerMenu
    ? headerShowButtons < $headerMenuItemsStore?.length
      ? $headerMenuItemsStore.slice(0, headerShowButtons)
      : $headerMenuItemsStore
    : [];

  $: headerDropMenuItems =
    headerMenu && headerShowButtons <= $headerMenuItemsStore?.length
      ? $headerMenuItemsStore.slice(
          headerShowButtons,
          $headerMenuItemsStore.length
        )
      : [];

  $: defaultQuery = QueryUtils.buildQuery(filter);
  $: queryExtension = QueryUtils.buildQuery(searchFilter);
  $: query = brain.extendQuery(defaultQuery, [queryExtension]);

  // Fetch data
  $: fetch = brain.createFetch($dataSourceStore);
  $: fetch.update({
    query,
    sortColumn,
    sortOrder,
    limit,
    paginate,
  });

  // Tree context
  $: treeOptions.set({
    ...$$props,
    treeType,
    rootButtons,
    nodeIcon,
    nodeSelection: true,
    groupSelectable,
    groupMenuDropItems: $groupMenuItemsStore,
    groupNodeIcon,
    groupFields,
    groupNodeLabel,
    selectedNodes,
    selectedGroups,
    menuStore,
    checkboxes,
    nodeMenuIcon,
    nodeMenuDropItems: $nodeMenuItemsStore,
    chevronPosition,
    hasSlot,
    quiet,
    showCount,
  });

  $: definition = $fetch?.definition;
  $: recursive = treeType == "recursive" && joinField;
  $: hasSlot = $component.children;

  $: primaryDisplay = brain.getPrimaryDisplay(definition);
  $: idColumn = idColumn || definition?.primary[0];
  $: idColumnType = definition?.schema[idColumn]?.type || "string";

  $: brain.buildTree($fetch?.rows, $treeOptions);
  $: brain.loadSelections(selectedIds, idColumn, idColumnType);

  // Handle search as a nested component
  // $: handleSearch({ detail: $parentFilter });

  $: actions = [
    {
      type: ActionTypes.RefreshDatasource,
      callback: () => fetch.refresh(),
    },
  ];

  $: list = controlType == "list";

  // Generate our context data
  $: context = {
    row: inBuilder ? $fetch?.rows[0] || {} : $currentRow, // Dynamically enriched at execution time
    breadcrumbs: brain.getBreadcrumbs(
      $selectedAncestors,
      $selectedNodes,
      rootNodes
    ),
    selected: $selectedRows,
    selectedGroups: $selectedGroups,
    selectedIds: $selectedNodes.length ? $selectedNodes : [],
    selectedAncestors: $selectedAncestors,
    selectedDescendants: $selectedDescendants,
  };

  const filterTree = async function (tree, searchLabel) {
    $filtering = true;
    await tick();

    function resetVisibility(node) {
      node.visible = false;
      node.children.forEach((child) => resetVisibility(child));
    }

    function applyFilter(node) {
      const labelStr = String(node.label || "").toLowerCase();
      const searchStr = searchLabel.toLowerCase();
      const matchesLabel = labelStr.includes(searchStr);

      node.children.forEach((child) => applyFilter(child));

      if (matchesLabel) {
        const startIndex = labelStr.indexOf(searchStr);
        const endIndex = startIndex + searchStr.length;
        const originalLabel = String(node.label || "");
        node.label = `${originalLabel.slice(0, startIndex)}<span style="font-size: bigger;  padding: 0.1rem 0rem; background-color: var(--highlighter-bg); border-radius: 2px;">${originalLabel.slice(startIndex, endIndex)}</span>${originalLabel.slice(endIndex)}`;
      } else {
        node.label = String(node.label || ""); // Reset to original string
      }

      const hasVisibleChildren = node.children.some((child) => child.visible);
      node.visible = matchesLabel || hasVisibleChildren;
      node.open = matchesLabel || hasVisibleChildren;
    }

    tree.forEach((root) => {
      resetVisibility(root);
      applyFilter(root);
    });

    hasMatches = tree.some((t) => t.visible);
    rootNodes = rootNodes;
    $filtering = false;
    return;
  };

  async function clearFilter(tree) {
    function resetNode(node) {
      node.visible = true;
      node.open = false;
      node.color = undefined;
      node.label = String(node.label).replace(
        /<span[^>]*>(.*?)<\/span>/g,
        "$1"
      ); // Remove highlight markup
      node.children.forEach((child) => resetNode(child));
    }

    tree.forEach((root) => resetNode(root));
    hasMatches = false;
    rootNodes = rootNodes;
    return;
  }

  setContext("superTreeOptions", treeOptions);
  setContext("superTreeFilter", treeFilter);

  $: $component.styles = {
    ...$component.styles,
    normal: {
      display: "flex",
      overflow: "hidden",
      ...$component.styles.normal,
    },
  };
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<div
  class="super-tree"
  class:flex
  class:quiet
  class:disabled
  class:nested
  class:list
  class:with-header={header && !nested}
  class:rootless={!rootless}
  use:styleable={$component.styles}
>
  <Provider {actions} data={context} />

  {#if header}
    <TreeHeader
      headerText={headerText || datasource?.label}
      {headerButtons}
      {headerDropMenuItems}
      {headerMenuIcon}
      {quiet}
      {searchable}
      {inBuilder}
      on:search={brain.handleSearch}
    />
  {/if}

  {#if $fetch.loading && !$fetch.loaded}
    <div class="loader" class:list>
      <div class="animation"></div>
    </div>
  {:else}
    <div class="tree">
      {#if rootless}
        {#if rootNodes.some((node) => node.visible)}
          {#each rootNodes as node, idx}
            <Tree
              {...node}
              children={node.children}
              {disabled}
              {quiet}
              {list}
              flat={!recursive && !groupFields?.length}
              {selectedNodes}
              {selectedGroups}
              on:nodeSelect={brain.handleNodeSelect}
              on:nodeAction={brain.handleNodeAction}
            >
              <slot></slot>
            </Tree>
          {/each}
        {:else}
          <div class="tree-node">
            <div class="tree-node-item empty">No matching nodes found</div>
          </div>
        {/if}
      {:else}
        <Tree
          id={"tree-root"}
          {disabled}
          hasSlot={$component.children}
          renderSlot={false}
          label={rootNodeName || datasource?.label || $component.name}
          children={rootNodes}
          open={!collapsed || hasMatches}
          {quiet}
          {list}
          flat={!recursive}
          on:nodeSelect={brain.handleNodeSelect}
          on:nodeAction={brain.handleNodeAction}
        >
          <slot />
        </Tree>
      {/if}

      {#if slotPosition == "after" && $component.children}
        <slot />
      {/if}
    </div>
  {/if}
</div>

<style>
  .super-tree {
    height: 360px;
    width: 14rem;
    overflow: hidden;
    color: var(--spectrum-global-color-gray-700);
    display: flex;
    flex-direction: column;
    position: relative;
    background-color: var(--spectrum-global-color-gray-100);
    --selected-bg: var(--spectrum-global-color-gray-300);
    --hover-bg: var(--spectrum-global-color-gray-200);
    --highlighter-bg: rgba(0, 255, 0, 0.25);
  }

  .super-tree.quiet {
    border-color: transparent;
    background-color: transparent;
    --selected-bg: var(--spectrum-global-color-gray-200);
    --hover-bg: var(--spectrum-global-color-gray-100);
  }

  .super-tree.quiet :global(.tree-node) {
    font-weight: 400;
  }

  .super-tree.quiet :global(.tree-node > .tree-node-item.selected) {
    background-color: var(--spectrum-global-color-gray-200);
  }

  .super-tree.nested {
    flex: none;
    width: 100%;
    border: none;
    height: unset;
    min-height: fit-content;
    min-width: fit-content;
    background-color: transparent;
  }

  .super-tree :global(.tree-node) {
    position: relative;
    width: 100%;
    line-height: 1.75rem;
    background: unset;
    padding: unset;
    border: unset;
    color: var(--spectrum-global-color-gray-700);
  }

  .super-tree :global(.tree-node > .tree-node-item) {
    position: relative;
    display: flex;
    justify-content: flex-start;
    align-items: center;
    max-height: 1.75rem;
    overflow: hidden;
    background-color: transparent;
    border-radius: 0.25rem;
  }

  .super-tree :global(.tree-node > .tree-node-item.selected) {
    background-color: var(--selected-bg);
    color: var(--spectrum-global-color-gray-800) !important;
  }

  .super-tree :global(.tree-node > .tree-node-item.selected .children-count) {
    color: var(--spectrum-global-color-gray-800) !important;
  }

  .super-tree :global(.tree-node > .tree-node-item.hasChildren:not(.disabled)) {
    cursor: pointer;
  }

  .super-tree :global(.tree-node > .tree-node-item.selectable:hover) {
    cursor: pointer;
  }

  .super-tree
    :global(
      .tree-node
        > .tree-node-item:hover:not(.selected):not(.disabled):not(.is-menu-open)
    ) {
    background-color: var(--hover-bg);
    color: var(--spectrum-global-color-gray-800);
  }

  .super-tree :global(.tree-node > .tree-node-item:hover:not(.disabled)),
  .super-tree :global(.tree-node > .tree-node-item.is-menu-open) {
    color: var(--spectrum-global-color-gray-700);
    background-color: var(--selected-bg);
  }

  .super-tree
    :global(.tree-node > .tree-node-item:hover:not(.disabled) .children-count),
  .super-tree
    :global(.tree-node > .tree-node-item.is-menu-open .children-count) {
    color: var(--spectrum-global-color-gray-800) !important;
  }

  .super-tree
    :global(.tree-node > .tree-node-item:hover:not(.disabled) > .menu-icon),
  .super-tree :global(.tree-node > .tree-node-item.is-menu-open > .menu-icon) {
    visibility: visible;
  }

  .super-tree :global(.tree-node > .tree-node-item > .menu-icon) {
    visibility: hidden;
    cursor: pointer;
    color: var(--spectrum-global-color-gray-700);
  }

  .super-tree :global(.tree-node > .tree-node-item.rightChevron) {
    padding-left: 0.75rem;
  }

  .super-tree :global(.tree-node > .tree-node-item.rightChevron.hasDropMenu) {
    padding-left: 0.25rem !important;
  }

  .super-tree :global(.tree-node > .tree-node-item.empty) {
    color: var(--spectrum-global-color-gray-600);
    font-style: italic;
    padding-left: 0.75rem;
  }

  .super-tree :global(.tree-node > .tree) {
    position: relative;
    margin-left: 1rem;
    display: flex;
    min-width: 190px;
    flex-direction: column;
    align-items: stretch;
  }

  .super-tree > .tree {
    flex: auto;
    overflow: auto;
    display: flex;
    flex-direction: column;
    align-items: stretch;
  }

  .super-tree.disabled {
    color: var(--spectrum-global-color-gray-600);
  }

  .super-tree.list > .tree {
    gap: 0.25rem;
    padding: 0.5rem;
  }

  .super-tree.list.nested > .tree {
    padding: unset;
  }

  .super-tree.list :global(.tree-node) {
    background-color: var(--spectrum-global-color-gray-75);
    border: 1px solid var(--spectrum-global-color-gray-300);
    border-radius: 4px;
  }

  .super-tree.list :global(.tree-node:hover) {
    border-color: var(--spectrum-global-color-gray-400);
  }

  .super-tree.list :global(.tree-node.selected) {
    border-color: var(--spectrum-global-color-blue-500);
  }

  .super-tree.list :global(.tree-node > .tree-node-item) {
    padding: 0.25rem;
    max-height: unset;
  }

  .super-tree.list :global(.tree-node > .tree) {
    padding: 0.5rem;
    gap: 0.5rem;
    margin-left: 1rem;
  }

  .super-tree.flex:not(.nested) {
    flex: auto;
  }

  .loader {
    display: flex;
    align-items: center;
    padding-left: 1rem;
    height: 2rem;
  }

  .loader.list {
    height: 2.6rem;
  }

  .loader > .animation {
    height: 5px;
    aspect-ratio: 5;
    -webkit-mask: linear-gradient(90deg, #0000, #000 20% 80%, #0000);
    mask: linear-gradient(90deg, #0000, #000 20% 80%, #0000);
    background: radial-gradient(
        closest-side at 37.5% 50%,
        var(--spectrum-global-color-blue-500) 94%,
        #0000
      )
      0 / calc(80% / 3) 100%;
    animation: l48 0.75s infinite linear;
  }

  @keyframes l48 {
    100% {
      background-position: 36.36%;
    }
  }
</style>
