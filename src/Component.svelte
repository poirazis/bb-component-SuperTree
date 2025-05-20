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
    enrichButtonActions,
    QueryUtils,
    memo,
    API,
    builderStore,
    screenStore,
    notificationStore,
  } = getContext("sdk");

  const component = getContext("component");
  const parentTree = getContext("superTreeOptions");
  const allContext = getContext("context");

  const lookupComponent = (components, id) => {
    let parent;
    let pos = components?.findIndex((comp) => comp._id == id);
    if (pos > -1) return components[pos];
    else
      components?.forEach((comp) => {
        if (!parent) parent = lookupComponent(comp._children, id);
      });

    return parent;
  };

  const parent = lookupComponent(
    $screenStore.activeScreen.props._children,
    $component.path.at(-2)
  );

  const nested = parent?._component == "plugin/bb-component-SuperTree";

  export let branchName;
  export let controlType = "tree";
  export let treeType = "flat";
  export let chevronPosition = "left";
  export let collapsed;
  export let quiet;

  export let searchMode;
  export let disabled;
  export let rootless;

  export let header;
  export let headerText;
  export let headerMenu;
  export let headerMenuItems;
  export let headerShowButtons;
  export let headerMenuIcon = "ri-more-2-fill";

  export let nodeIcon;
  export let nodeIconTemplate;
  export let nodeMenu;
  export let nodeMenuIcon = "ri-more-fill";
  export let nodeShowButtons;
  export let nodeMenuItems = [];
  export let nodeSelection;
  export let nodeFGColor;
  export let nodeBGColor;
  export let nodeIconColor;

  export let checkboxes;
  export let maxNodeSelection;

  export let groupMenu;
  export let groupNodeIcon;
  export let groupNodeLabel;
  export let groupSelectable;
  export let groupMenuItems = [];
  export let groupShowButtons = 0;
  export let groupFGColor;
  export let groupBGColor;
  export let groupSlot;

  export let datasource;
  export let filter = [];
  export let sortColumn;
  export let sortOrder;
  export let limit;
  export let paginate;
  export let labelColumn;
  export let labelTemplate;
  export let structuredData;

  export let joinField;
  export let groupFields = [];

  export let flex;
  export let idColumn;

  // Events
  export let onNodeSelect;
  export let onNodeClick;

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
  let selectedRows = new writable([]);
  let menuStore = new writable({});
  let query = {};
  let defaultQuery;
  let searchFilter;
  let filtering = writable(false);

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

  $: nodeButtons = nodeMenu
    ? nodeShowButtons < $nodeMenuItemsStore?.length
      ? $nodeMenuItemsStore.slice(0, nodeShowButtons)
      : $nodeMenuItemsStore
    : [];
  $: nodeMenuDropItems =
    nodeMenu && nodeShowButtons <= $nodeMenuItemsStore?.length
      ? $nodeMenuItemsStore.slice(nodeShowButtons, $nodeMenuItemsStore.length)
      : [];

  $: groupButtons = groupMenu
    ? groupShowButtons < $groupMenuItemsStore?.length
      ? $groupMenuItemsStore.slice(0, groupShowButtons)
      : $groupMenuItemsStore
    : [];

  $: groupMenuDropItems = groupMenu
    ? groupShowButtons < $groupMenuItemsStore?.length
      ? $groupMenuItemsStore.slice(
          groupShowButtons,
          $groupMenuItemsStore.length
        )
      : []
    : [];

  $: defaultQuery = QueryUtils.buildQuery(filter);
  $: queryExtension = QueryUtils.buildQuery(searchFilter);
  $: query = extendQuery(defaultQuery, [queryExtension]);

  // Fetch data
  $: fetch = createFetch($dataSourceStore);
  $: fetch.update({
    query,
    sortColumn,
    sortOrder,
    limit,
    paginate,
  });

  $: resetSelections(nodeSelection);

  // Tree context
  $: treeOptions.set({
    ...$$props,
    treeType,
    nodeIcon,
    nodeSelection,
    groupSelectable,
    groupMenu,
    groupButtons,
    groupMenuDropItems,
    groupNodeIcon,
    groupFields,
    groupNodeLabel,
    selectedNodes,
    menuStore,
    checkboxes,
    nodeMenu,
    nodeMenuIcon,
    nodeButtons,
    nodeMenuDropItems,
    chevronPosition,
    hasSlot,
    quiet,
  });

  $: definition = $fetch?.definition;
  $: recursive = treeType == "recursive" && joinField;
  $: hasSlot = $component.children;

  $: primaryDisplay = getPrimaryDisplay(definition);

  $: buildTree($fetch?.rows, $treeOptions);

  $: actions = [
    {
      type: ActionTypes.RefreshDatasource,
      callback: () => fetch.refresh(),
    },
  ];

  $: list = controlType == "list";
  $: context = {
    selected: $selectedRows,
    selectedIds: $selectedNodes,
    selectedPath: $selectedNodes.length
      ? getAncestors(rootNodes, $selectedNodes[0])
      : [],
  };

  const createFetch = (datasource) => {
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
  };

  const extendQuery = (defaultQuery, extensions) => {
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
  };

  const enrichNode = (row, idx, options = {}) => {
    const {
      isGroup = false,
      groupValue,
      type = "node",
      children = [],
      visible = true,
    } = options;

    return {
      id: row[idColumn] || `${groupValue}`,
      renderSlot: hasSlot,
      type,
      group: isGroup ? groupValue : undefined,
      visible,
      quiet: isGroup ? quiet : undefined,
      row: isGroup ? undefined : row,
      open: $builderStore.inBuilder && $component.children && idx === 0,
      icon: nodeIconTemplate
        ? processStringSync(nodeIconTemplate, { [comp_id]: { ...row } })
        : nodeIcon,
      iconColor: nodeIconColor
        ? processStringSync(nodeIconColor, { [comp_id]: { ...row } })
        : undefined,
      label: labelTemplate
        ? processStringSync(labelTemplate, { [comp_id]: { ...row } })
        : row[labelColumn || primaryDisplay],
      color: nodeFGColor
        ? processStringSync(nodeFGColor, { [comp_id]: { ...row } })
        : undefined,
      bgColor: nodeBGColor
        ? processStringSync(nodeBGColor, { [comp_id]: { ...row } })
        : undefined,
      children,
    };
  };

  // Initialize Tree Structure
  const buildTree = (rows, filter) => {
    let nodes = [];
    groupByValues.clear();
    rootNodes = [];

    if (structuredData) {
      rootNodes = rows;
      return;
    }

    if (treeType === "groupBy" && groupFields?.length > 0) {
      rootNodes = buildGroupNodes(rows, groupFields, 0);
    } else if (recursive) {
      rows
        .filter((x) => !x[joinField])
        .forEach((row, idx) => {
          rootNodes.push({
            ...enrichNode(row, idx, { children: getChildNodes(row, rows) }),
          });
        });
    } else {
      rows.forEach((row, idx) =>
        rootNodes.push({
          ...enrichNode(row, idx),
        })
      );
    }
  };

  const buildGroupNodes = (rows, groupFields, level) => {
    const currentField = groupFields[level];
    const nextLevel = level + 1;
    const isLastLevel = nextLevel >= groupFields.length;

    const groupValues = new Set(rows.map((row) => row[currentField]).sort());

    return Array.from(groupValues).map((value, idx) => {
      const groupRows = rows.filter((row) => row[currentField] === value);
      const children = isLastLevel
        ? getGroupChildNodes(value, groupRows)
        : buildGroupNodes(groupRows, groupFields, nextLevel);

      return {
        type: "groupBranch",
        selectable: groupSelectable,
        renderSlot: groupSlot,
        id: `${currentField}-${value}`,
        open: $builderStore.inBuilder && $component.children && idx === 0,
        icon: groupNodeIcon,
        label: groupNodeLabel
          ? processStringSync(groupNodeLabel, {
              ...$allContext,
              [comp_id]: {
                ...$allContext[comp_id],
                group: value,
              },
            })
          : value,
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
  };

  const getGroupChildNodes = (groupValue, groupRows) => {
    return groupRows.map((row, idx) =>
      enrichNode(row, idx, {
        isGroup: true,
        groupValue,
        type: "groupItem",
        children: getChildNodes(row),
      })
    );
  };

  const getChildNodes = (row, rows) => {
    let children = [];

    if (recursive) {
      rows
        .filter((x) => x[joinField] == row[idColumn])
        .forEach((row, idx) => {
          children.push({
            ...enrichNode(row, idx, {
              children: getChildNodes(row, rows),
              visible: true,
              type: "node",
            }),
            quiet,
          });
        });
    }
    return children;
  };

  const handleNodeSelect = async (e) => {
    if (inBuilder) return;

    let row = e.detail.row;
    let index = $selectedNodes.findIndex((x) => x == e.detail.id);

    if (index > -1) row = {};

    if (index > -1) {
      $selectedNodes.splice(index, 1);
      $selectedNodes = $selectedNodes;
    } else if ($selectedNodes.length < maxNodeSelection) {
      $selectedNodes = [...$selectedNodes, e.detail.id];
    } else if (maxNodeSelection == 1) {
      $selectedNodes = [e.detail.id];
    } else {
      notificationStore.actions.warning(
        "Cannot select more than " + maxNodeSelection + " items"
      );
    }
    $selectedRows = $fetch?.rows?.filter((row) =>
      $selectedNodes.find((x) => x == row[idColumn])
    );

    let cmd = enrichButtonActions(onNodeSelect, {
      ...$allContext,
      [comp_id]: {
        ...$allContext[comp_id],
        ...row,
      },
    });

    await cmd?.();
  };

  const handleNodeClick = async (e) => {
    let cmd = enrichButtonActions(onNodeClick, {
      ...$allContext,
      [comp_id]: { ...context, ...e.detail.row },
    });
    await cmd?.();
  };

  const handleNodeAction = async (e) => {
    let cmd = enrichButtonActions(e.detail.onClick, {
      ...$allContext,
      [comp_id]: { ...context, ...e.detail, group: e.detail.group },
    });

    await cmd?.();
  };

  const handleSearch = (e) => {
    if ($filtering) return;

    if (searchMode == "client") {
      if (e.detail) {
        filterTree(rootNodes, e.detail);
      } else clearFilter(rootNodes);
      return;
    } else {
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
  };

  const resetSelections = () => {
    $selectedNodes = [];
  };

  const getPrimaryDisplay = (definition) => {
    if (definition) {
      if (definition.primaryDisplay) return definition.primaryDisplay;
      else return Object.keys(definition.schema)[0];
    }
  };

  setContext("superTreeOptions", treeOptions);

  $: $component.styles = {
    ...$component.styles,
    normal: {
      flex: flex ? "auto" : "none",
      display: "flex",
      overflow: "hidden",
      height: nested ? "auto" : "360",
      ...$component.styles.normal,
    },
  };

  const filterTree = async function (tree, searchLabel) {
    $filtering = true;
    await tick();

    function resetVisibility(node) {
      node.visible = false;
      node.children.forEach((child) => resetVisibility(child));
    }

    function applyFilter(node) {
      const matchesLabel = node.label
        .toLowerCase()
        .includes(searchLabel.toLowerCase());

      node.children.forEach((child) => applyFilter(child));

      if (matchesLabel) {
        node.color = "var(--spectrum-global-color-yellow-400)";
      } else {
        node.color = null;
      }

      const hasVisibleChildren = node.children.some((child) => child.visible);
      node.visible = matchesLabel || hasVisibleChildren;
      node.open = matchesLabel || hasVisibleChildren;
    }

    tree.forEach((root) => {
      resetVisibility(root);
      applyFilter(root);
    });

    $filtering = false;
    return;
  };

  async function clearFilter(tree) {
    function resetNode(node) {
      node.visible = true;
      node.open = false;
      node.color = undefined;
      node.children.forEach((child) => resetNode(child));
    }

    tree.forEach((root) => resetNode(root));
    rootNodes = rootNodes;
    $selectedNodes = [];
    return;
  }

  function getAncestors(tree, nodeId) {
    function findAncestors(node, targetId, ancestors = []) {
      if (node.id === targetId) {
        return ancestors;
      }

      for (const child of node.children) {
        const result = findAncestors(child, targetId, [
          ...ancestors,
          node.label,
        ]);
        if (result) {
          return result;
        }
      }

      return null;
    }

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
  }
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<div use:styleable={$component.styles}>
  <div class="super-tree" class:quiet class:disabled class:nested class:list>
    <Provider {actions} data={context} />
    {#if header && !nested}
      <TreeHeader
        headerText={headerText || datasource?.label}
        {headerButtons}
        {headerDropMenuItems}
        {headerMenuIcon}
        {quiet}
        {searchable}
        {inBuilder}
        on:search={handleSearch}
      />
    {/if}
    {#if ($fetch.loading && !$fetch.loaded) || $filtering}
      <div class="loader" class:list>
        <div class="animation" />
      </div>
    {:else}
      <div class="tree">
        {#if rootNodes.length}
          {#if rootless}
            {#each rootNodes as node, idx}
              {#if node.visible !== false}
                <Tree
                  {...node}
                  {disabled}
                  {quiet}
                  {list}
                  flat={!recursive}
                  on:nodeSelect={handleNodeSelect}
                  on:nodeClick={handleNodeClick}
                  on:nodeAction={handleNodeAction}
                >
                  <slot />
                </Tree>
              {/if}
            {/each}
          {:else}
            <Tree
              id={"tree-root"}
              {disabled}
              hasSlot={$component.children}
              renderSlot={false}
              label={branchName || datasource?.label || $component.name}
              children={rootNodes}
              open={!collapsed}
              {quiet}
              {list}
              flat={!recursive}
              on:nodeSelect={handleNodeSelect}
              on:nodeClick={handleNodeClick}
              on:nodeAction={handleNodeAction}
            >
              <slot />
            </Tree>
          {/if}
        {:else}
          <div class="tree-node">
            <div class="tree-node-item empty">
              <span>No Records Found</span>
            </div>
          </div>
        {/if}
      </div>
    {/if}
  </div>
</div>

<style>
  .super-tree {
    flex: auto;
    min-height: 360px;
    width: 240px;
    overflow: hidden;
    color: var(--spectrum-global-color-gray-700);
    display: flex;
    flex-direction: column;
    position: relative;

    &.quiet {
      border-color: transparent;
      background-color: transparent;

      & * .tree-node {
        font-weight: 400;

        & > .tree-node-item {
          &.selected {
            background-color: var(--spectrum-global-color-gray-200);
          }
        }
      }
    }

    &.nested {
      width: 100%;
      height: auto;
      border: unset;
      min-height: unset;
    }

    & * .tree-node {
      position: relative;
      width: 100%;
      line-height: 1.75rem;
      font-size: 13px;

      & > .tree-node-item {
        position: relative;
        display: flex;
        justify-content: flex-start;
        max-height: 1.75rem;
        overflow: hidden;
        background-color: transparent;

        &.selected {
          background-color: var(--spectrum-global-color-blue-100);
          color: var(--spectrum-global-color-gray-900);
        }

        &.flat {
          padding-left: 0.5rem;
        }

        &.hasChildren:not(.disabled) {
          cursor: pointer;
        }

        &:hover:not(.selected):not(.disabled) {
          background-color: var(--spectrum-global-color-gray-200);
        }

        &:hover:not(.disabled),
        &.is-menu-open {
          color: var(--spectrum-global-color-gray-900);
          & > .menu-icon {
            visibility: visible;
          }
        }

        & > .menu-icon {
          visibility: hidden;
          cursor: pointer;
          color: var(--spectrum-global-color-gray-600);
        }

        &.rightChevron {
          padding-left: 0.75rem;
        }

        &.empty {
          color: var(--spectrum-global-color-gray-600);
          font-style: italic;
          padding-left: 0.75rem;
        }
      }

      & > .tree {
        position: relative;
        margin-left: 1.25rem;
        display: flex;
        min-width: 240px;
        flex-direction: column;
        align-items: stretch;
      }
    }
    & > .tree {
      flex: auto;
      overflow: auto;
      display: flex;
      flex-direction: column;
      align-items: stretch;
    }

    &.disabled {
      color: var(--spectrum-global-color-gray-600);
    }

    &.list {
      & > .tree {
        gap: 0.5rem;
        padding: 0.5rem;
      }

      &.nested {
        & > .tree {
          padding: unset;
        }
      }

      & > * .tree-node {
        background-color: var(--spectrum-global-color-gray-75);
        border: 1px solid var(--spectrum-global-color-gray-300);
        border-radius: 4px;

        &:hover {
          border-color: var(--spectrum-global-color-gray-400);
        }

        &.selected {
          border-color: var(--spectrum-global-color-blue-500);
        }

        & > .tree-node-item {
          padding: 0.25rem;
          max-height: unset;
        }

        & > .tree {
          padding: 0.5rem;
          gap: 0.5rem;
          margin-left: 1rem;
        }
      }
    }
  }

  .loader {
    display: flex;
    align-items: center;
    padding-left: 1rem;
    height: 2rem;

    &.list {
      height: 2.6rem;
    }

    & > .animation {
      height: 5px;
      aspect-ratio: 5;
      -webkit-mask: linear-gradient(90deg, #0000, #000 20% 80%, #0000);
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
  }
</style>
