<script>
  import { getContext } from "svelte"
  import { writable } from "svelte/store"
  import Tree from "../lib/Tree.svelte";
  import Skeleton from "./Skeleton.svelte";

  const { styleable, fetchData, processStringSync, LuceneUtils, API, builderStore, notificationStore } = getContext("sdk")
  const component = getContext("component")

  export let source 
  export let customNodes
  export let branchName

  export let collapsed
  export let quiet
  export let disabled
  export let rootless
  export let rootIcon
  export let nodeIcon
  export let groupNodeIcon
  export let groupNodeLabel
  export let nodeSelection
  export let maxNodeSelection

  export let datasource
  export let filter = {}
  export let sortColumn
  export let sortOrder
  export let limit
  export let paginate
  export let labelColumn
  export let valueColumn
  export let groupBy
  export let recursive
  export let joinField
  export let groupField
  export let linkFields

  // Events
  export let onNodeSelect
  export let onNodeClick

  let groupByValues = new Set();
  let rootNodes = []
  let selectedNodes = new writable([])
  let queryExtensions = {}

  $: defaultQuery = LuceneUtils.buildLuceneQuery(filter)
  $: query = extendQuery(defaultQuery, queryExtensions)

  // Fetch data and refresh when needed
  $: fetch = createFetch( source == "data" ? datasource : null )
  $: fetch.update({
    query,
    sortColumn,
    sortOrder,
    limit,
    paginate,
  })
  $: definition = $fetch?.definition
  $: primaryDisplay = definition?.primaryDisplay
  $: getRootNodes ($fetch?.rows, groupNodeLabel)

  const createFetch = datasource => {
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
    })
  }

  const extendQuery = (defaultQuery, extensions) => {
    const extensionValues = Object.values(extensions || {})
    let extendedQuery = { ...defaultQuery }
    extensionValues.forEach(extension => {
      Object.entries(extension || {}).forEach(([operator, fields]) => {
        extendedQuery[operator] = {
          ...extendedQuery[operator],
          ...fields,
        }
      })
    })
    return extendedQuery
  }

  const getRootNodes = rows => {
    let nodes = []
    groupByValues.clear()
    rootNodes = []

    if ( source == "custom" ) {
      rootNodes = customNodes.map( ( node, idx ) => {
        return {
          id: node.value, 
          label:node.label,
          hasSlot : $component.children,
          renderSlot: true,
          icon: nodeIcon, 
          nodeSelection: nodeSelection,
          selectedNodes: selectedNodes,
          children: [],
          open: $builderStore.inBuilder && $component.children && idx == 0,
        }
      })
    }

    if ( groupBy ) {
      rows.map( row => { groupByValues.add(row[groupField]) })
      nodes = [...groupByValues]
      rootNodes = nodes.map( (x, idx ) => { 
        return {
          type: "group",
          renderSlot: false,
          hasSlot : $component.children,
          nodeSelection: nodeSelection,
          selectedNodes: selectedNodes,
          id: x,
          open: $builderStore.inBuilder && $component.children && idx == 0,
          icon: groupNodeIcon, 
          label: processStringSync( groupNodeLabel || "{{ value }}" , { value : x } ),
          children : getGroupChildNodes(x)} })
    } else {
      rows.map( (row, idx )=> { rootNodes = [ ...rootNodes, 
        { 
          id: row[valueColumn],
          renderSlot: true,
          hasSlot : $component.children,
          nodeSelection: nodeSelection,
          selectedNodes: selectedNodes,
          icon: nodeIcon, 
          open: $builderStore.inBuilder && $component.children && idx == 0,
          label : row[labelColumn || primaryDisplay],
          children: getChildNodes(row)
        }]  
      })
    }
    if ( recursive && joinField ) {
      rows.map( row => {
        rootNodes = [ ...rootNodes, { 
          icon: nodeIcon, 
          renderSlot: true,
          hasSlot : $component.children,
          label : row[labelColumn || primaryDisplay]}]
      })
    }    
  }

  const getGroupChildNodes = groupValue => {
    return $fetch.rows.filter( row => row[groupField] == groupValue ).map ( (node,idx) => {
      return {
        id: node[valueColumn],
        icon: nodeIcon,
        renderSlot: true,
        quiet,
        hasSlot : $component.children,
        open: $builderStore.inBuilder && $component.children && idx == 0,
        label: node[labelColumn || primaryDisplay],
        nodeSelection: nodeSelection,
        selectedNodes: selectedNodes,
        children : getChildNodes( node )
      }
    })
  }

  const getChildNodes = row => {
    let children = []
    if ( linkFields?.length ) {
      linkFields.forEach(element => {
        if (validLinkField(element)) children.push( {
          type: "link",
          renderSlot : false,
          icon: "ri-link",
          label: element,
          quiet,
          nodeSelection: nodeSelection,
          selectedNodes: selectedNodes,
          children: row[element]?.map( link => { return { 
            nodeSelection: nodeSelection, 
            selectedNodes: selectedNodes,
            id: link["_id"],
            quiet,
            label: link.primaryDisplay } })
        })
        
      });
    }
    return children
  }

  const validLinkField = field => {
    return definition.schema[field]?.type == "link"
  }

  const handleNodeSelect = ( e ) => {
    let index = $selectedNodes.findIndex( x => x.id == e.detail.id )
    if ( index > -1 ) {
      $selectedNodes.splice(index,1)
      $selectedNodes = $selectedNodes
    } else if ( $selectedNodes.length < maxNodeSelection ) {
      $selectedNodes = [ ...$selectedNodes, e.detail ]
    } else if ( maxNodeSelection == 1 ) {
      $selectedNodes = [ e.detail ]
    } else {
      notificationStore.actions.warning("Cannot select more than " + maxNodeSelection + " items")
    }
    onNodeSelect?.( e.detail )
  }

  const handleNodeClick = ( e ) => {
    onNodeClick?.( e.detail )
  }
</script>
  <div use:styleable={$component.styles} >
    <ul class="spectrum-TreeView"
    class:spectrum-TreeView--quiet={quiet}
    style:visibility={"visible"}
    style:height={ $fetch?.loading ? "2rem" : $component.styles.normal.height ?? "auto"}
    style:overflow-y={"auto"}
    >
      {#if $fetch?.loading}
        <Skeleton> Loading </Skeleton>
      {:else if $fetch?.loaded}
        {#if rootless}
          {#each [...rootNodes] as {id, icon , label, renderSlot, children, open}, idx}
            <Tree 
              {id}
              {icon}
              {disabled}
              {nodeSelection}
              hasSlot={$component.children}
              {renderSlot}
              label={label || "Not Set"} 
              {open}
              {children}
              {selectedNodes}
              on:nodeSelect={handleNodeSelect}
              on:nodeClick={handleNodeClick}
              {quiet}
              > 
              <slot/>
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
            label={branchName || $component.name} 
            children={rootNodes} 
            open={!rootless && !collapsed || ($builderStore.inBuilder && $component.children)}
            {quiet}
            on:nodeSelect={handleNodeSelect}
            on:nodeClick={handleNodeClick}
            > 
            <slot />
          </Tree>
        {/if}
      {/if}
    </ul>
  </div>


<style>
  .spectrum-TreeView {
    min-width: 250px;
    margin: unset;
  }
</style>