<script>
  import { getContext, setContext } from "svelte";
  import { writable } from "svelte/store";
  import Tree from "../lib/Tree.svelte";
  import Skeleton from "./Skeleton.svelte";
  import CellString from "../../bb_super_components_shared/src/lib/SuperTableCells/CellString.svelte";
  import SuperPopover from "../../bb_super_components_shared/src/lib/SuperPopover/SuperPopover.svelte";

  const {
    styleable,
    ActionTypes,
    Provider,
    ContextScopes,
    fetchData,
    processStringSync,
    QueryUtils,
    memo,
    API,
    BlockComponent,
    Block,
    builderStore,
    notificationStore,
  } = getContext("sdk");

  const component = getContext("component");

  export let branchName;
  export let treeType = "list";

  export let collapsed;
  export let quiet;
  export let buttons;
  export let searchable;
  export let disabled;
  export let rootless;
  export let header;
  export let headerText = "New Super Tree";
  export let actionMenu;
  export let menuIcon;
  export let menuItems;
  export let rootIcon;

  export let nodeIcon;
  export let nodeMenu;
  export let nodeMenuIcon = "ri-more-fill";
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
  export let labelColumn;
  export let valueColumn;

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
  let query = {};
  let defaultQuery;
  let searchFilter;
  let hover;
  let inEdit = false;
  let openMenu = false;
  let menuAnchor;

  $: dataSourceStore.set(datasource);
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

  $: definition = $fetch?.definition;
  $: primaryDisplay = definition?.primaryDisplay;
  $: buildTree(
    $fetch?.rows,
    treeType,
    valueColumn,
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
              id: row[valueColumn],
              renderSlot: true,
              hasSlot: $component.children,
              open: $builderStore.inBuilder && $component.children && idx == 0,
              label: row[labelColumn || primaryDisplay],
              children: getChildNodes(row, rows),
            },
          ];
        });
    } else {
      rows.map((row, idx) => {
        rootNodes = [
          ...rootNodes,
          {
            id: row[valueColumn],
            renderSlot: true,
            hasSlot: $component.children,
            open: $builderStore.inBuilder && $component.children && idx == 0,
            label: row[labelColumn || primaryDisplay || valueColumn],
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
          id: node[valueColumn],
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
        .filter((x) => x[joinField] == row[valueColumn])
        .forEach((node) => {
          children.push({
            renderSlot: true,
            id: node[valueColumn],
            label: node[labelColumn],
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

  const searchTree = (node, term) => {};

  const filterTree = (term) => {};

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
    onNodeSelect?.(index < 0 ? e.detail : undefined);
  };

  const handleNodeClick = (e) => {
    onNodeClick?.(e.detail);
  };

  const handleSearch = (e) => {
    // For non recursive trees the filtering is done server side
    if (e.detail) {
      searchFilter = [
        {
          field: valueColumn,
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

  // Expose the Super Tree Context fot the Child nodes and nested Trees
  $: treeOptions.set({
    nodeIcon,
    nodeSelection,
    selectableBranches,
    selectedNodes,
    checkboxes,
    nodeMenu,
    nodeMenuIcon,
    nodeMenuItems,
  });

  $: cellOptions = {
    placeholder: "Search...",
    disabled,
    padding: "0.5rem",
    icon: "ri-search-line",
    debounce: 200,
    clearValueIcon: true,
    role: "inlineInput",
  };

  $: context = {
    selected: $selectedNodes,
    nodeLabel: "poi",
  };

  setContext("superTreeOptions", treeOptions);
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<div use:styleable={$component.styles}>
  <Provider {actions} data={context} scope={ContextScopes.Global}>
    {#if header}
      <div class="searchHeader">
        {#if (searchable && hover && !disabled) || searchFilter?.length || inEdit}
          <div
            on:mouseleave={() => (hover = false)}
            style:display={"flex"}
            style:justify-content={"stretch"}
            style:flex={"auto"}
            style:border-bottom={"2px solid var(--spectrum-global-color-gray-300)"}
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
            {headerText || datasource?.label}
          </div>
        {/if}

        {#if actionMenu && menuItems?.length}
          {#if menuItems.length > 1}
            <button
              class="spectrum-ActionButton spectrum-ActionButton--sizeM spectrum-ActionButton--quiet"
              class:is-selected={openMenu}
              bind:this={menuAnchor}
              on:click|stopPropagation={handleMenu}
            >
              <i class={menuIcon} />
            </button>
          {:else}
            <Block>
              <!-- svelte-ignore a11y-click-events-have-key-events -->
              {#each menuItems as { text, icon, disabled, onClick, quiet }}
                <BlockComponent
                  type="plugin/bb-component-SuperButton"
                  props={{
                    size: "M",
                    icon,
                    text,
                    quiet: true,
                    disabled,
                    onClick,
                  }}
                ></BlockComponent>
              {/each}
            </Block>
          {/if}
        {/if}

        <SuperPopover
          open={openMenu}
          anchor={menuAnchor}
          on:close={() => (openMenu = false)}
        >
          {#if menuItems?.length}
            <!-- svelte-ignore a11y-click-events-have-key-events -->
            <div class="actionMenu" on:click|stopPropagation={handleMenu}>
              <Block>
                {#each menuItems as { text, icon, disabled, onClick, quiet }}
                  <BlockComponent
                    type="plugin/bb-component-SuperButton"
                    props={{
                      size: "M",
                      icon,
                      text,
                      quiet: true,
                      disabled,
                      onClick,
                    }}
                  ></BlockComponent>
                {/each}
              </Block>
            </div>
          {/if}
        </SuperPopover>
      </div>
    {/if}

    <ul
      class="spectrum-TreeView"
      class:spectrum-TreeView--quiet={quiet}
      style:visibility={"visible"}
      style:height={"auto"}
      style:overflow-y={"auto"}
    >
      {#if $fetch.loading && !$fetch.loaded}
        <Skeleton>Loading</Skeleton>
      {:else if $fetch?.loaded && rootNodes.length}
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
          <!-- svelte-ignore a11y-invalid-attribute -->
          <a href="#" class="spectrum-TreeView-itemLink"> No Records Found </a>
        </li>
      {/if}
    </ul>
  </Provider>
</div>

<style>
  .spectrum-TreeView {
    flex: 1 1 auto;
    margin: unset;
    overflow-y: auto;
  }

  .searchHeader {
    min-width: 200px;
    height: 2rem;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.5rem;
    transition: all 230ms;
  }

  .actionMenu {
    width: 100%;
    display: flex;
    flex-direction: column;
    align-items: flex-end;
  }

  .title {
    font-size: 13px;
    font-weight: 700;
    letter-spacing: 0.72px;
    text-transform: uppercase;
    color: var(--spectrum-global-color-gray-700);
    transition: all 230ms;
    padding-left: 0.25rem;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow-x: hidden;
  }
</style>
