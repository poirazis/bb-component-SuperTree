<script>
  import { getContext, createEventDispatcher } from "svelte";
  import SuperPopover from "../../bb_super_components_shared/src/lib/SuperPopover/SuperPopover.svelte";
  import SuperButton from "../../bb_super_components_shared/src/lib/SuperButton/SuperButton.svelte";

  const { Provider, enrichButtonActions } = getContext("sdk");
  const treeOptions = getContext("superTreeOptions");
  const allContext = getContext("context");

  const dispatch = createEventDispatcher();

  export let label;
  export let children = [];
  export let quiet;
  export let open = false;
  export let hasSlot;
  export let renderSlot;
  export let id;
  export let disabled;

  let openMenu;
  let menuAnchor;
  let hover;

  $: context = {
    id,
    label,
    children,
  };

  $: if (disabled) open = false;

  $: selectionStore = $treeOptions.selectedNodes;
  $: menuStore = $treeOptions.menuStore;
  $: selected = $selectionStore.findIndex((x) => x.id == id) > -1;
  $: icon = $treeOptions.nodeIcon;

  const handleClick = (e) => {
    if (disabled) return;

    if (children?.length || hasSlot) {
      open = !open;
      return;
    }

    if ($treeOptions.nodeSelection) {
      dispatch("nodeSelect", { id, label });
      return;
    }

    dispatch("nodeClick", { id, label });
  };

  const handleSelect = (e) => {
    if ($treeOptions.nodeSelection) {
      dispatch("nodeSelect", { id, label });
      return;
    }
  };
</script>

<!-- svelte-ignore a11y-missing-attribute -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
<li
  class="spectrum-TreeView-item"
  class:is-disabled={disabled}
  class:is-selected={selected}
  class:is-open={open}
  on:click|stopPropagation={handleSelect}
>
  <a
    class="spectrum-TreeView-itemLink"
    style:padding-left={children?.length ? "0.25rem" : "1.5rem"}
    on:mouseenter={() => (hover = true)}
    on:mouseleave={() => (hover = false)}
  >
    {#if children?.length}
      <i
        class="ri-arrow-right-s-line chevron"
        class:open
        class:childless={children?.length < 1}
        on:click|self|stopPropagation={handleClick}
      >
      </i>
    {/if}

    {#if $treeOptions?.checkboxes}
      {#if selected}
        <i
          on:click|preventDefault|stopPropagation={handleSelect}
          class="ri-checkbox-fill icon"
        />
      {:else}
        <i
          on:click|preventDefault|stopPropagation={handleSelect}
          class="ri-checkbox-blank-line icon"
        />
      {/if}
    {/if}

    {#if icon}
      <i class={icon} class:icon />
    {/if}
    {#if $treeOptions?.nodeButtons}
      <div on:mousedown={() => ($menuStore = id)}>
        {#each $treeOptions?.nodeButtons as { text, icon, disabled, quiet, onClick }}
          <SuperButton
            size="S"
            {icon}
            {text}
            {quiet}
            {disabled}
            onClick={enrichButtonActions(onClick, $allContext)}
          />
        {/each}
      </div>
    {/if}

    {label || "Not Set"}

    <!-- The Node Action Menu  -->
    {#if (hover || openMenu) && $treeOptions.nodeMenuDropItems?.length}
      <div>
        <SuperButton
          bind:anchor={menuAnchor}
          size="S"
          icon={$treeOptions.nodeMenuIcon}
          fillOnHover
          quiet
          onClick={(e) => {
            openMenu = !openMenu;
            $menuStore = openMenu ? id : false;
          }}
          text=""
        />
      </div>

      <SuperPopover
        open={openMenu}
        align={"left"}
        anchor={menuAnchor}
        on:close={() => {
          openMenu = false;
        }}
      >
        {#if $treeOptions?.nodeMenuDropItems?.length}
          <!-- svelte-ignore a11y-click-events-have-key-events -->
          <!-- svelte-ignore a11y-no-static-element-interactions -->
          <div
            class="actionMenu"
            on:click={(e) => {
              openMenu = false;
            }}
          >
            {#each $treeOptions?.nodeMenuDropItems as { text, icon, disabled, onClick }}
              <SuperButton
                size="S"
                {icon}
                {text}
                quiet
                {disabled}
                onClick={enrichButtonActions(onClick, $allContext)}
                menuItem
                menuAlign="left"
              />
            {/each}
          </div>
        {:else}
          <p>No Actions Defined</p>
        {/if}
      </SuperPopover>
    {/if}
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
  }
  .spectrum-TreeView-itemLink {
    width: 100%;
    display: flex;
    justify-content: flex-start;
    gap: 0.25rem;
  }

  .icon {
    font-size: 16px;
    color: var(--spectrum-global-color-gray-600);
    z-index: 1;
  }
  .chevron {
    transition: all 130ms;
    font-size: 16px;
    color: var(--spectrum-global-color-gray-500);
    z-index: 2;
  }

  .chevron.open {
    transform: rotate(90deg);
    color: var(--spectrum-global-color-gray-800);
  }

  .actionMenu {
    min-width: 100px;
    display: flex;
    flex-direction: column;
    align-items: stretch;
  }
</style>
