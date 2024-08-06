<script>
  import { getContext, setContext } from "svelte";
  import { writable } from "svelte/store";

  import Tree from "../lib/Tree.svelte";
  import Skeleton from "./Skeleton.svelte";
  import CellString from "../../bb_super_components_shared/src/lib/SuperTableCells/CellString.svelte";
  import SuperPopover from "../../bb_super_components_shared/src/lib/SuperPopover/SuperPopover.svelte";
  import SuperButton from "../../bb_super_components_shared/src/lib/SuperButton/SuperButton.svelte";

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
    enrichButtonActions,
  } = getContext("sdk");

  const component = getContext("component");
  const fullContext = getContext("context");

  export let branchName;
  export let treeType = "list";

  export let collapsed;
  export let quiet;

  export let searchable;
  export let disabled;
  export let rootless;

  export let header;
  export let headerText = "New Super Tree";
  export let actionMenu;
  export let showButtons;
  export let menuIcon = "ri-more-fill";
  export let menuItems;
  export let rootIcon;

  export let nodeIcon;
  export let nodeMenu;
  export let nodeMenuIcon = "ri-more-fill";
  export let nodeShowButtons;
  export let nodeMenuItems;
  export let groupNodeIcon;

  export let groupNodeLabel;
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

  export let idColumn;
  export let labelColumn;
  export let labelTemplate;

  export let joinField;
  export let groupField;
  export let linkFields;

  // Events
  export let onNodeSelect;
  export let onNodeClick;

  const dataSourceStore = memo(datasource);
  const treeOptions = memo({});

  let groupByValues = new Set();
  let rootNodes = [];
  let selectedNodes = new writable([]);
  let menuStore = new writable({});
  let query = {};
  let defaultQuery;
  let searchFilter;
  let hover;
  let inEdit = false;
  let openMenu = false;
  let menuAnchor;

  $: headerButtons =
    showButtons < menuItems?.length
      ? menuItems.slice(0, showButtons)
      : menuItems;

  $: headerMenuItems =
    showButtons < menuItems?.length
      ? menuItems.slice(showButtons, menuItems.length)
      : [];

  $: nodeButtons =
    nodeShowButtons < nodeMenuItems?.length
      ? nodeMenuItems.slice(0, nodeShowButtons)
      : nodeMenuItems;

  $: nodeMenuDropItems =
    nodeShowButtons < nodeMenuItems?.length
      ? nodeMenuItems.slice(nodeShowButtons, nodeMenuItems.length)
      : [];

  $: dataSourceStore.set(datasource);
  $: defaultQuery = QueryUtils.buildQuery(filter);
  $: queryExtension = QueryUtils.buildQuery(searchFilter);
  $: query = extendQuery(defaultQuery, [queryExtension]);
  $: if (
    $builderStore.inBuilder &&
    $component.selected &&
    !joinField &&
    auto_join_field
  ) {
    builderStore.actions.updateProp("joinField", auto_join_field);
    builderStore.actions.updateProp("treeType", "recursive");
  }

  // Fetch data and refresh when needed
  $: fetch = createFetch($dataSourceStore);
  $: fetch.update({
    query,
    sortColumn,
    sortOrder,
    limit,
    paginate,
  });

  $: parseJoinField($fetch?.rows, joinField, treeType);

  $: definition = $fetch?.definition;
  $: auto_join_field = detectJoinField(definition);
  $: primaryDisplay = definition?.primaryDisplay;
  $: buildTree(
    $fetch?.rows,
    treeType,
    labelColumn,
    joinField,
    groupField,
    groupNodeLabel
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
    selectableBranches,
    selectedNodes,
    menuStore,
    checkboxes,
    nodeMenu,
    nodeMenuIcon,
    nodeMenuItems,
    nodeButtons,
    nodeMenuDropItems,
  });

  $: cellOptions = {
    placeholder: "Search...",
    disabled,
    padding: "0.5rem",
    debounce: 200,
    clearValueIcon: true,
    role: "inlineInput",
  };

  $: menuId = $menuStore;
  $: menuRow = menuId ? $fetch.rows.find((e) => e._id == menuId) : {};

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
  const buildTree = (rows, filter) => {
    let nodes = [];
    groupByValues.clear();
    rootNodes = [];

    if (treeType == "groupBy") {
      rows.map((row) => {
        groupByValues.add(row[groupField]);
      });
      nodes = [...groupByValues];
      rootNodes = nodes.map((x, idx) => {
        return {
          type: "group",
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
    } else if (treeType == "recursive") {
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
              label: row[primaryDisplay],
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
            label: row[primaryDisplay],
            children: getChildNodes(row),
          },
        ];
      });
    }
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
        if (validLinkField(element))
          children.push({
            type: "link",
            renderSlot: false,
            label: element,
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

    if (treeType == "recursive" && joinField) {
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

  const handleMenu = (e) => {
    openMenu = !openMenu;
  };

  const parseJoinField = (rows, field) => {
    if (treeType == "recursive" && field && rows?.length)
      rows
        .filter((x) => x[field])
        .filter((x) => typeof x[field] == "string")
        .map((row) => (row[field] = safeParse(row[field])));
  };

  const detectJoinField = (schema) => {
    return Object.keys(schema?.schema || {}).find((e) => e.endsWith("_self_"));
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
<div use:styleable={$component.styles}>
  <Provider {actions} data={context}>
    {#if header}
      <div
        class="searchHeader"
        style:border-bottom={inEdit
          ? "1px solid var(--spectrum-global-color-blue-400)"
          : hover
            ? "2px solid var(--spectrum-global-color-gray-200)"
            : "2px solid transparent"}
      >
        {#if (searchable && hover && !disabled) || searchFilter?.length || inEdit}
          <div
            on:mouseleave={() => (hover = false)}
            style:display={"flex"}
            style:justify-content={"stretch"}
            style:flex={"auto"}
          >
            <CellString
              {cellOptions}
              on:change={handleSearch}
              on:enteredit={() => (inEdit = true)}
              on:exitedit={() => (inEdit = searchFilter?.length)}
            />
          </div>
        {:else}
          <div
            class="title"
            on:mouseenter={() => (hover = searchable)}
            on:mouseleave={() => (hover = false)}
          >
            {#if searchable}
              <i
                class="ri-search-line"
                style="font-size: 12px; color: var(--spectrum-global-color-gray-500)"
              />
            {/if}
            {headerText || datasource?.label}
          </div>
        {/if}

        {#if actionMenu && menuItems?.length}
          <div>
            {#each headerButtons as { text, icon, disabled, onClick, quiet }}
              <SuperButton
                size="M"
                {icon}
                {disabled}
                {quiet}
                onClick={enrichButtonActions(onClick, $fullContext)}
                {text}
              />
            {/each}

            {#if headerMenuItems.length}
              <SuperButton
                bind:anchor={menuAnchor}
                size="M"
                icon={menuIcon}
                quiet
                selected={openMenu}
                onClick={handleMenu}
                text=""
              />
            {/if}
          </div>
        {/if}
      </div>
    {/if}

    {#if $fetch?.loading && !$fetch?.loaded}
      <div style="height: 2rem;">
        <Skeleton></Skeleton>
      </div>
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
            {#each [...rootNodes] as { id, icon, label, renderSlot, children, open }, idx}
              <Tree
                {id}
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
              on:nodeSelect={handleNodeSelect}
              on:nodeClick={handleNodeClick}
            >
              <slot />
            </Tree>
          {/if}
        {:else}
          <li class="spectrum-TreeView-item">
            <p class="spectrum-TreeView-itemLink">No Records Found</p>
          </li>
        {/if}
      </ul>
    {/if}
    <!-- Header Action Menu Popover -->
    <SuperPopover
      open={openMenu}
      anchor={menuAnchor}
      on:close={() => (openMenu = false)}
    >
      {#if headerMenuItems?.length}
        <div
          class="actionMenu"
          on:click={(e) => {
            openMenu = false;
          }}
        >
          {#each headerMenuItems as { text, icon, disabled, onClick, quiet }}
            <SuperButton
              size="M"
              {icon}
              {disabled}
              quiet
              menuItem={true}
              menuAlign="right"
              onClick={enrichButtonActions(onClick, $fullContext)}
              {text}
            />
          {/each}
        </div>
      {/if}
    </SuperPopover>
  </Provider>
</div>

<style>
  .spectrum-TreeView {
    flex: 1 1 auto;
    margin: unset;
    overflow-y: auto;
  }

  .searchHeader {
    width: 100%;
    height: 2rem;
    display: flex;
    align-items: center;
    justify-content: space-between;
    transition: all 130ms;
  }

  .actionMenu {
    min-width: 120px;
    display: flex;
    flex-direction: column;
    align-items: stretch;
  }

  .title {
    font-size: 13px;
    font-weight: 700;
    letter-spacing: 0.72px;
    text-transform: uppercase;
    color: var(--spectrum-global-color-gray-700);
    transition: all 230ms;
    padding-left: 0.35rem;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow-x: hidden;
    display: flex;
    align-items: center;
    gap: 0.65rem;
  }
</style>
