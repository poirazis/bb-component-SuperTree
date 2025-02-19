<script>
  import TreeHeader from "../lib/TreeHeader.svelte";
  import { getContext, setContext } from "svelte";
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

  export let searchable;
  export let disabled;
  export let rootless;

  export let header;
  export let headerText;
  export let headerMenu;
  export let headerMenuItems;
  export let headerShowButtons;
  export let headerMenuIcon = "ri-more-2-fill";
  export let collapsible;

  export let nodeIcon;
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

  export let joinField;
  export let groupField;

  export let flex;
  export let idColumn;

  // Events
  export let onNodeSelect;
  export let onNodeClick;

  // Use Stores for non Primitive Data types to avoid unecessary refreshes
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
  let clientHeight;
  let loaded;

  $: comp_id = $component.id;
  $: inBuilder = $builderStore.inBuilder;
  $: quiet = $parentTree ? $parentTree.quiet : quiet;

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

  // Fetch data and refresh when needed
  $: fetch = createFetch($dataSourceStore);
  $: fetch.update({
    query,
    sortColumn,
    sortOrder,
    limit,
    paginate,
  });

  $: resetSelections(nodeSelection);

  // Expose the Super Tree Context fot the Child nodes and nested Trees
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
    groupField,
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

  $: buildTreeAsync($fetch?.rows, $treeOptions);

  $: actions = [
    {
      type: ActionTypes.RefreshDatasource,
      callback: () => fetch.refresh(),
    },
  ];

  $: list = controlType == "list";
  $: context = {
    selected: $selectedRows,
    selectedIds: $selectedRows.map((x) => x[idColumn]),
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

  // Initialize Tree Structure
  const buildTree = async (rows, filter) => {
    let nodes = [];
    groupByValues.clear();
    rootNodes = [];

    if (treeType == "groupBy" && groupField) {
      rows.map((row) => {
        groupByValues.add(row[groupField]);
      });
      groupByValues = new Set(Array.from(groupByValues).sort());
      nodes = [...groupByValues];
      rootNodes = nodes.map((x, idx) => {
        return {
          type: "groupBranch",
          selectable: groupSelectable,
          renderSlot: groupSlot,
          id: x,
          open: $builderStore.inBuilder && $component.children && idx == 0,
          icon: groupNodeIcon,
          label: groupNodeLabel
            ? processStringSync(groupNodeLabel, {
                ...$allContext,
                [comp_id]: {
                  ...$allContext[comp_id],
                  group: x,
                },
              })
            : x,
          color: groupFGColor
            ? processStringSync(groupFGColor, {
                ...$allContext,
                [comp_id]: {
                  ...$allContext[comp_id],
                  group: x,
                },
              })
            : undefined,
          bgColor: groupBGColor
            ? processStringSync(groupBGColor, {
                ...$allContext,
                [comp_id]: {
                  ...$allContext[comp_id],
                  group: x,
                },
              })
            : undefined,
          children: getGroupChildNodes(x),
        };
      });
    } else if (recursive) {
      rows
        .filter((x) => !x[joinField])
        .map((row, idx, arr) => {
          rootNodes.push({
            id: row[idColumn],
            renderSlot: hasSlot,
            icon: nodeIcon,
            row,
            open: $builderStore.inBuilder && $component.children && idx == 0,
            label: labelTemplate
              ? processStringSync(labelTemplate, { [comp_id]: { ...row } })
              : row[primaryDisplay],
            color: nodeFGColor
              ? processStringSync(nodeFGColor, { [comp_id]: { ...row } })
              : undefined,
            bgColor: nodeBGColor
              ? processStringSync(nodeBGColor, { [comp_id]: { ...row } })
              : undefined,
            iconColor: nodeIconColor
              ? processStringSync(nodeIconColor, { [comp_id]: { ...row } })
              : undefined,
            children: getChildNodes(row, rows),
          });
        });
    } else {
      rows.map((row, idx) => {
        rootNodes = [
          ...rootNodes,
          {
            id: row[idColumn],
            row,
            renderSlot: hasSlot,
            open: $builderStore.inBuilder && $component.children && idx == 0,
            icon: nodeIcon,
            label: labelTemplate
              ? processStringSync(labelTemplate, { [comp_id]: { ...row } })
              : row[primaryDisplay],
            color: nodeFGColor
              ? processStringSync(nodeFGColor, { [comp_id]: { ...row } })
              : undefined,
            bgColor: nodeBGColor
              ? processStringSync(nodeBGColor, { [comp_id]: { ...row } })
              : undefined,
            iconColor: nodeIconColor
              ? processStringSync(nodeIconColor, { [comp_id]: { ...row } })
              : undefined,
            children: getChildNodes(row),
          },
        ];
      });
    }
  };

  const buildTreeAsync = async (rows) => {
    loaded = false;
    await buildTree(rows);
    loaded = true;
  };

  const getGroupChildNodes = (groupValue) => {
    return $fetch.rows
      .filter((row) => row[groupField] == groupValue)
      .map((row, idx) => {
        return {
          id: row[idColumn],
          renderSlot: hasSlot,
          type: "groupItem",
          group: groupValue,
          quiet,
          row,
          open: $builderStore.inBuilder && $component.children && idx == 0,
          icon: nodeIcon,
          label: labelTemplate
            ? processStringSync(labelTemplate, { [comp_id]: { ...row } })
            : row[primaryDisplay],
          color: nodeFGColor
            ? processStringSync(nodeFGColor, { [comp_id]: { ...row } })
            : undefined,
          bgColor: nodeBGColor
            ? processStringSync(nodeBGColor, { [comp_id]: { ...row } })
            : undefined,
          children: getChildNodes(row),
        };
      });
  };

  const getChildNodes = (row, rows) => {
    let children = [];

    if (recursive) {
      rows
        .filter((x) => x[joinField] == row[idColumn])
        .forEach((row) => {
          children.push({
            renderSlot: hasSlot,
            id: row[idColumn],
            icon: nodeIcon,
            row,
            label: labelTemplate
              ? processStringSync(labelTemplate, { [comp_id]: { ...row } })
              : row[primaryDisplay],
            color: nodeFGColor
              ? processStringSync(nodeFGColor, { [comp_id]: { ...row } })
              : undefined,
            bgColor: nodeBGColor
              ? processStringSync(nodeBGColor, { [comp_id]: { ...row } })
              : undefined,
            quiet,
            children: getChildNodes(row, rows),
          });
        });
    }
    return children;
  };

  const handleNodeSelect = async (e) => {
    if (inBuilder) return;

    let row = e.detail.row;

    let index = $selectedNodes.findIndex((x) => x.id == e.detail.id);

    // If unselecting clear the context
    if (index > -1) row = {};

    if (index > -1) {
      $selectedNodes.splice(index, 1);
      $selectedNodes = $selectedNodes;
    } else if ($selectedNodes.length < maxNodeSelection) {
      $selectedNodes = [...$selectedNodes, e.detail];
    } else if (maxNodeSelection == 1) {
      $selectedNodes = [e.detail];
    } else {
      notificationStore.actions.warning(
        "Cannot select more than " + maxNodeSelection + " items"
      );
    }
    $selectedRows = $fetch?.rows?.filter((row) =>
      $selectedNodes.find((x) => x.id == row[idColumn])
    );

    // Enrich the context with the values of the row being selected
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

  /**
   *
   * @param e.detail will contain the onClick event, the row and the group ( if any )
   *
   * We enrich our own context with the current row before enriching and
   * executing the action
   */
  const handleNodeAction = async (e) => {
    let cmd = enrichButtonActions(e.detail.onClick, {
      ...$allContext,
      [comp_id]: { ...context, ...e.detail.row, group: e.detail.group },
    });
    await cmd?.();
  };

  const handleSearch = (e) => {
    // For non recursive trees the filtering is done server side
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
      "background-color": quiet
        ? "transparent"
        : "var(--spectrum-global-color-gray-50)",
      flex: flex ? "auto" : "none",
      display: "flex",
      overflow: "hidden",
      height: nested ? "auto" : "360",
      ...$component.styles.normal,
    },
  };
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<div use:styleable={$component.styles}>
  <div
    bind:clientHeight
    class="super-tree"
    class:quiet
    class:disabled
    class:nested
    class:list
  >
    <Provider {actions} data={context} />
    {#if header && !nested}
      <TreeHeader
        headerText={headerText || datasource?.label}
        {headerButtons}
        {headerDropMenuItems}
        {headerMenuIcon}
        {quiet}
        {searchable}
        {collapsible}
        {inBuilder}
        on:search={handleSearch}
      />
    {/if}

    {#if $fetch.loading && !$fetch.loaded}
      <div class="loader" class:list>
        <div class="animation" />
      </div>
    {:else}
      <div class="tree">
        {#if rootNodes.length}
          {#if rootless}
            {#each rootNodes as node, idx (idx)}
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
            {#if nested}
              <div
                class="tree-node-item disabled"
                style:padding-left={"1.25rem"}
              >
                <span>
                  {branchName || datasource?.label || $component.name}</span
                >
              </div>
            {:else}
              <span class="tree-node-item flat disabled">No Records Found</span>
            {/if}
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
    border: 1px solid var(--spectrum-global-color-gray-200);
    display: flex;
    flex-direction: column;
    position: relative;
    &.quiet {
      border-color: transparent;

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
      line-height: 2rem;

      & > .tree-node-item {
        position: relative;
        display: flex;
        justify-content: flex-start;
        max-height: 2rem;
        overflow: hidden;
        background-color: transparent;
        color: var(--spectrum-global-color-gray-700);

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
          background-color: var(--spectrum-global-color-gray-75);
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
      }

      & > .tree {
        position: relative;
        margin-left: 1.25rem;
        display: flex;
        flex-direction: column;
        align-items: stretch;
      }
    }
    & > .tree {
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
