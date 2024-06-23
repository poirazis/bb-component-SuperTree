<script>
  import { getContext, createEventDispatcher } from "svelte";
  const { Provider } = getContext("sdk");
  const treeOptions = getContext("superTreeOptions");

  const dispatch = createEventDispatcher();

  export let label;
  export let children = [];
  export let quiet;
  export let open = false;
  export let hasSlot;
  export let renderSlot;
  export let id;
  export let disabled;

  $: context = {
    nodeId: id,
    nodeLabel: label,
  };

  $: if (disabled) open = false;
  $: selectionStore = $treeOptions.selectedNodes;
  $: selected = $selectionStore.findIndex((x) => x.id == id) > -1;
  $: icon = $treeOptions.nodeIcon;

  const handleClick = (e) => {
    if (disabled) return;

    if (children?.length || hasSlot) {
      open = !open;
      return;
    } else if ($treeOptions.nodeSelection) {
      dispatch("nodeSelect", { id, label });
      return;
    }
    dispatch("nodeClick", { id, label });
  };
</script>

<!-- svelte-ignore a11y-missing-attribute -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<li
  class="spectrum-TreeView-item"
  class:is-disabled={disabled}
  class:is-selected={selected}
  class:is-open={open}
>
  <a
    class="spectrum-TreeView-itemLink"
    on:click={handleClick}
    style:padding-left={children?.length ? "0.25rem" : "1.75rem"}
  >
    {#if children?.length}
      <i
        class="ri-arrow-right-s-line chevron"
        class:open
        class:childless={children?.length < 1}
      >
      </i>
    {/if}

    {#if $treeOptions?.checkboxes}
      {#if selected}
        <i class="ri-checkbox-line icon" />
      {:else}
        <i class="ri-checkbox-blank-line icon" />
      {/if}
    {/if}

    {#if icon}
      <i class={icon} class:icon />
    {/if}

    {label || "Not Set"}
  </a>

  {#if children?.length || hasSlot}
    <ul
      class="spectrum-TreeView"
      style:display={open ? "block" : "none"}
      class:spectrum-TreeView--quiet={quiet}
    >
      {#each children as node, idx}
        <svelte:self
          id={node.id}
          hasSlot={node.hasSlot}
          renderSlot={node.renderSlot}
          label={node.label}
          children={node.children}
          quiet={node.quiet}
          open={node.open}
          on:nodeSelect
          on:nodeClick
        >
          <slot />
        </svelte:self>
      {/each}
    </ul>
    {#if renderSlot}
      <Provider data={context}>
        <slot />
      </Provider>
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

  .icon {
    font-size: 18px;
    color: var(--spectrum-global-color-gray-700);
  }
  .chevron {
    transition: all 130ms;
    font-size: 16px;
    color: var(--spectrum-global-color-gray-700);
  }

  .chevron.open {
    transform: rotate(90deg);
    color: var(--spectrum-global-color-gray-800);
  }
</style>
