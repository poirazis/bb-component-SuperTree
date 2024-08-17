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
    QueryUtils,
    memo,
    API,
    builderStore,
    notificationStore,
  } = getContext("sdk");

  const component = getContext("component");

  export let branchName;
  export let controlType = "tree";
  export let treeType = "flat";
  export let collapsed;
  export let quiet;

  export let searchable;
  export let disabled;
  export let rootless;

  export let header;
  export let headerText;
  export let actionMenu;
  export let showButtons;
  export let menuIcon = "ri-more-fill";
  export let menuItems = [];
  export let rootIcon;

  export let nodeIcon;
  export let nodeMenu;
  export let nodeMenuIcon = "ri-more-fill";
  export let nodeShowButtons;
  export let nodeMenuItems = [];
  export let groupNodeIcon;

  export let groupNodeLabel;
  export let groupSelectable;
  export let groupMenuItems = [];
  export let groupShowButtons = 0;
  export let nodeSelection;
  export let checkboxes;
  export let selectableBranches;
  export let maxNodeSelection;

  export let datasource;
  export let filter = [];
  export let sortColumn;
  export let sortOrder;
  export let limit;
  export let paginate;

  export let idColumn = "_id";
  export let labelColumn;
  export let labelTemplate;

  export let joinField;
  export let listjoinField;
  export let groupField;
  export let linkFields;

  // Events
  export let onNodeSelect;
  export let onNodeClick;

  // Use Stores for non Primitive Data types to avoid unecessary refreshes
  const dataSourceStore = memo(datasource);
  $: dataSourceStore.set(datasource);

  const treeOptions = memo({});
  const headerMenuItemsStore = memo(menuItems);
  $: headerMenuItemsStore.set(menuItems);

  const nodeMenuItemsStore = memo(nodeMenuItems);
  $: nodeMenuItemsStore.set(nodeMenuItems);

  const groupMenuItemsStore = memo(groupMenuItems);
  $: groupMenuItemsStore.set(groupMenuItems);

  const linkFieldsStore = memo(linkFields);
  $: linkFieldsStore.set(linkFields);

  let groupByValues = new Set();
  let rootNodes = [];
  let selectedNodes = new writable([]);
  let menuStore = new writable({});
  let query = {};
  let defaultQuery;
  let searchFilter;
  let buildingTree = true;

  $: headerButtons =
    showButtons < $headerMenuItemsStore?.length
      ? $headerMenuItemsStore.slice(0, showButtons)
      : $headerMenuItemsStore;

  $: headerMenuItems =
    showButtons <= $headerMenuItemsStore?.length
      ? $headerMenuItemsStore.slice(showButtons, $headerMenuItemsStore.length)
      : [];

  $: nodeButtons =
    nodeShowButtons < $nodeMenuItemsStore?.length
      ? $nodeMenuItemsStore.slice(0, nodeShowButtons)
      : $nodeMenuItemsStore;

  $: nodeMenuDropItems =
    nodeShowButtons <= $nodeMenuItemsStore?.length
      ? $nodeMenuItemsStore.slice(nodeShowButtons, $nodeMenuItemsStore.length)
      : [];

  $: groupButtons =
    groupShowButtons < $groupMenuItemsStore?.length
      ? $groupMenuItemsStore.slice(0, groupShowButtons)
      : $groupMenuItemsStore;

  $: groupMenuDropItems =
    groupShowButtons < $groupMenuItemsStore?.length
      ? $groupMenuItemsStore.slice(
          groupShowButtons,
          $groupMenuItemsStore.length
        )
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

  $: parseJoinField($fetch?.rows, joinField, recursive);

  $: definition = $fetch?.definition;
  $: recursive = treeType == "recursive" && joinField;

  $: primaryDisplay = definition?.primaryDisplay;
  $: buildTreeAsync(
    $fetch?.rows,
    treeType,
    labelColumn,
    joinField,
    groupField,
    groupNodeLabel,
    recursive,
    $linkFieldsStore,
    $dataSourceStore
  );

  $: actions = [
    {
      type: ActionTypes.RefreshDatasource,
      callback: () => fetch.refresh(),
    },
  ];
  // Expose the Super Tree Context fot the Child nodes and nested Trees
  $: treeOptions.set({
    nodeIcon,
    nodeSelection,
    groupSelectable,
    groupMenuItems,
    groupButtons,
    groupMenuDropItems,
    selectedNodes,
    menuStore,
    checkboxes,
    nodeMenu,
    nodeMenuIcon,
    nodeMenuItems,
    nodeButtons,
    nodeMenuDropItems,
  });

  $: list = controlType == "list";
  $: accordion = controlType == "accordion";
  $: menuRow = $menuStore ? $fetch.rows.find((e) => e._id == $menuStore) : {};

  $: context = {
    selected: $selectedNodes,
    menuRow,
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
    return extendedQuery;
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
          type: "branch",
          selectable: groupSelectable,
          renderSlot: false,
          hasSlot: $component.children,
          id: x,
          open: $builderStore.inBuilder && $component.children && idx == 0,
          icon: groupNodeIcon,
          label: processStringSync(groupNodeLabel || "{{ value }}", {
            value: x,
          }),
          children: getGroupChildNodes(x),
        };
      });
    } else if (recursive) {
      rows
        .filter((x) => !x[joinField])
        .map((row, idx, arr) => {
          rootNodes = [
            ...rootNodes,
            {
              id: row[idColumn],
              renderSlot: true,
              hasSlot: $component.children,
              open: $builderStore.inBuilder && $component.children && idx == 0,
              label: labelTemplate
                ? processStringSync(labelTemplate, { Row: row })
                : row[primaryDisplay],
              children: getChildNodes(row, rows),
            },
          ];
        });
    } else {
      rows.map((row, idx) => {
        rootNodes = [
          ...rootNodes,
          {
            id: row[idColumn],
            renderSlot: true,
            hasSlot: $component.children,
            open: $builderStore.inBuilder && $component.children && idx == 0,
            label: labelTemplate
              ? processStringSync(labelTemplate, { Row: row })
              : row[primaryDisplay],
            children: getChildNodes(row),
          },
        ];
      });
    }
    buildingTree = false;
  };

  const buildTreeAsync = async (rows) => {
    await buildTree(rows);
  };

  const getGroupChildNodes = (groupValue) => {
    return $fetch.rows
      .filter((row) => row[groupField] == groupValue)
      .map((node, idx) => {
        return {
          id: node[idColumn],
          renderSlot: true,
          quiet,
          hasSlot: $component.children,
          open: $builderStore.inBuilder && $component.children && idx == 0,
          label: node[labelColumn || primaryDisplay],
          children: getChildNodes(node),
        };
      });
  };

  const getChildNodes = (row, rows) => {
    let children = [];

    if (linkFields?.length) {
      linkFields.forEach((element) => {
        if (validLinkField(element) && row[element])
          children.push({
            type: "branch",
            renderSlot: false,
            label: element,
            icon: "ri-links-line",
            quiet,
            nodeSelection: nodeSelection,
            selectedNodes: selectedNodes,
            children: row[element]?.map((link) => {
              return {
                nodeSelection: nodeSelection,
                selectedNodes: selectedNodes,
                id: link["_id"],
                quiet,
                label: link.primaryDisplay,
              };
            }),
          });
      });
    }

    if (recursive) {
      rows
        .filter((x) => x[joinField]?._id == row[idColumn])
        .forEach((node) => {
          children.push({
            renderSlot: true,
            id: node[idColumn],
            label: node[labelColumn || primaryDisplay],
            quiet,
            children: getChildNodes(node, rows),
          });
        });
    }
    return children;
  };

  const validLinkField = (field) => {
    return definition.schema[field]?.type == "link";
  };

  const handleNodeSelect = (e) => {
    let index = $selectedNodes.findIndex((x) => x.id == e.detail.id);
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
    let selectedRows = $fetch?.rows?.filter((row) =>
      $selectedNodes.find((x) => x.id == row._id)
    );

    onNodeSelect?.({ selectedRows });
  };

  const handleNodeClick = (e) => {
    onNodeClick?.(e.detail);
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
  const parseJoinField = (rows, field, recursive) => {
    if (recursive && field && rows?.length)
      rows
        .filter((x) => x[field])
        .filter((x) => typeof x[field] == "string")
        .map((row) => (row[field] = safeParse(row[field])));
  };

  const safeParse = (str) => {
    let res;
    try {
      res = JSON.parse(str)[0];
    } catch {}
    return res;
  };

  setContext("superTreeOptions", treeOptions);
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<div class="super-tree" use:styleable={$component.styles}>
  <Provider {actions} data={context}>
    {#if header}
      <TreeHeader
        headerText={headerText || datasource?.label}
        {headerButtons}
        {headerMenuItems}
        headerMenuIcon={menuIcon}
        {quiet}
        {list}
        {searchable}
        on:search={handleSearch}
      />
    {/if}

    {#if $fetch?.loading && !$fetch?.loaded}
      <div style="height: 2rem;"></div>
    {:else}
      <ul
        class="spectrum-TreeView"
        class:spectrum-TreeView--quiet={quiet}
        style:visibility={"visible"}
        style:height={"auto"}
        style:overflow-y={"auto"}
      >
        {#if rootNodes.length}
          {#if rootless}
            {#each [...rootNodes] as { id, icon, label, renderSlot, children, open, type, selectable }, idx}
              <Tree
                {id}
                {type}
                {selectable}
                {icon}
                {disabled}
                hasSlot={$component.children}
                {renderSlot}
                label={label || "Not Set"}
                {open}
                {children}
                on:nodeSelect={handleNodeSelect}
                on:nodeClick={handleNodeClick}
                {quiet}
                {list}
                {accordion}
                flat={!recursive}
              >
                <slot />
              </Tree>
            {/each}
          {:else}
            <Tree
              id={"tree-root"}
              icon={rootIcon}
              {nodeSelection}
              {selectedNodes}
              {disabled}
              hasSlot={$component.children}
              renderSlot={false}
              label={branchName || datasource?.label || $component.name}
              children={rootNodes}
              open={!rootless && !collapsed}
              {quiet}
              {list}
              {accordion}
              flat={!recursive}
              on:nodeSelect={handleNodeSelect}
              on:nodeClick={handleNodeClick}
            >
              <slot />
            </Tree>
          {/if}
        {:else}
          <li class="spectrum-TreeView-item">
            <!-- svelte-ignore a11y-invalid-attribute -->
            <a href="" class="spectrum-TreeView-itemLink">No Records Found</a>
          </li>
        {/if}
      </ul>
    {/if}
    <!-- Header Action Menu Popover -->
  </Provider>
</div>

<style>
  .super-tree {
    min-width: 220px;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    overflow-y: auto;
  }
  .spectrum-TreeView {
    margin: unset;
  }
</style>
