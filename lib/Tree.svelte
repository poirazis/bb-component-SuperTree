<script>
  import { getContext, createEventDispatcher } from "svelte"
  const { Provider } = getContext("sdk")
  const dispatch = createEventDispatcher();

  export let label
  export let children = []
  export let quiet
  export let open = false
  export let nodeSelection
  export let selectedNodes
  export let icon
  export let hasSlot
  export let renderSlot
  export let id
  export let disabled


  $: context = {
    nodeId : id,
    nodeLabel : label
  }

  $: if ( disabled ) open = false
  
  const handleClick = e => {
    if ( disabled ) return;

    if ( children?.length || hasSlot ) {
      open = !open 
      return
    }
    else if (nodeSelection) {
      dispatch("nodeSelect", {id, label})
    }

    dispatch("nodeClick", {id,label})
  }
</script>

<!-- svelte-ignore a11y-missing-attribute -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<li class="spectrum-TreeView-item"
  class:is-disabled={disabled}
  class:is-selected={$selectedNodes.findIndex( x => x.id == id ) > -1 }
  class:is-open={open} >
  <a 
    class="spectrum-TreeView-itemLink"
    style:padding-left={ children?.length || hasSlot || quiet ? "0.25rem" : "1rem" }
    on:click|stopPropagation={handleClick}
    >
    {#if (children?.length || (hasSlot &&  renderSlot) ) && !quiet} 
      <i class="ri-arrow-right-s-line chevron" class:open> </i>
    {/if}

    {#if icon}
      <i class={icon} class:icon/>
    {/if}

    {label || "Not Set"}
  </a>

  {#if open}
    {#if children?.length || hasSlot}
      <ul class="spectrum-TreeView" class:spectrum-TreeView--quiet={quiet}>
        {#each children as node, idx }
          <svelte:self 
            id={node.id}
            hasSlot={node.hasSlot}
            renderSlot={node.renderSlot}
            icon={node.icon}
            nodeSelection={node.nodeSelection}
            selectedNodes={node.selectedNodes}
            label={node.label}
            children={node.children}
            quiet={node.quiet}
            open={node.open}
            on:nodeSelect
            on:nodeClick
            > 
            <slot/>
          </svelte:self>
        {/each}
      </ul>
      {#if renderSlot}
        <Provider data={context}>
          <slot />
        </Provider>
      {/if}
    {/if}
  {/if}
</li>

<style>
  .spectrum-TreeView-item {
    transition: all 130ms;
    padding-left: 0.25rem;
  }

  .spectrum-TreeView-itemLink {
    width: 100%;
    display: flex;
    justify-content: stretch;
    gap: 0.5rem;
    padding-left: unset;
  }

  .spectrum-TreeView-itemLink:hover > i {
    color: var(--spectrum-global-color-blue-400);
  }

  .icon {
    font-size: 16px;
    color: var(--spectrum-global-color-gray-500);
  }

  .selected {
    font-size: 16px;
    color: var(--spectrum-global-color-blue-500);
  }
  .chevron {
    transition: all 130ms;
    font-size: 16px;
    color: var(--spectrum-global-color-gray-500);  
  }

  .chevron.open {
    transform: rotate(90deg);
    color: var(--spectrum-global-color-gray-800);
  }
</style>