<script>
  import { getContext, setContext } from "svelte";
  import { writable } from "svelte/store";
  import Tree from "../lib/Tree.svelte";
  import Skeleton from "./Skeleton.svelte";

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
    builderStore,
    notificationStore,
  } = getContext("sdk");

  const component = getContext("component");

  export let branchName;
  export let treeType = "list";

  export let collapsed;
  export let quiet;
  export let disabled;
  export let rootless;
  export let rootIcon;
  export let nodeIcon;
  export let groupNodeIcon;
  export let groupNodeLabel;
  export let nodeSelection;
  export let checkboxes;
  export let selectableBranches;
  export let maxNodeSelection;

  export let datasource;
  export let filter = {};
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

  $: dataSourceStore.set(datasource);

  let groupByValues = new Set();
  let rootNodes = [];
  let selectedNodes = new writable([]);
  let query = {};
  let defaultQuery;

  $: defaultQuery = QueryUtils.buildQuery(filter);
  $: query = extendQuery(defaultQuery, {});

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
  $: getRootNodes($fetch?.rows, groupNodeLabel);
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

  const getRootNodes = (rows) => {
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
          nodeSelection: nodeSelection,
          selectedNodes: selectedNodes,
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
              nodeSelection: nodeSelection,
              selectedNodes: selectedNodes,
              icon: nodeIcon,
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
          icon: nodeIcon,
          renderSlot: true,
          quiet,
          hasSlot: $component.children,
          open: $builderStore.inBuilder && $component.children && idx == 0,
          label: node[labelColumn || primaryDisplay],
          nodeSelection: nodeSelection,
          selectedNodes: selectedNodes,
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
            icon: nodeIcon,
            id: node[valueColumn],
            label: node[labelColumn],
            quiet,
            nodeSelection: nodeSelection,
            selectedNodes: selectedNodes,
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
    onNodeSelect?.(index < 0 ? e.detail : undefined);
  };

  const handleNodeClick = (e) => {
    onNodeClick?.(e.detail);
  };

  $: treeOptions.set({
    nodeIcon,
    nodeSelection,
    selectedNodes,
    checkboxes,
  });
  setContext("superTreeOptions", treeOptions);
</script>

<div use:styleable={$component.styles}>
  <Provider {actions} scope={ContextScopes.Global}>
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
    min-width: 250px;
    margin: unset;
    overflow-y: auto;
  }
</style>
