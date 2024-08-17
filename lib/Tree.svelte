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
  export let list;
  export let flat;
  export let type;
  export let icon;
  export let accordion;

  let openMenu;
  let menuAnchor;
  let hover;

  $: context = {
    id,
    label,
    children,
  };

  $: if (disabled) open = false;
  $: branch = type == "branch";

  $: selectionStore = $treeOptions.selectedNodes;
  $: menuStore = $treeOptions.menuStore;
  $: selected = $selectionStore.findIndex((x) => x.id == id) > -1;

  $: hasButtons = branch
    ? $treeOptions?.groupButtons.length
    : $treeOptions?.nodeButtons.length;

  $: buttons = branch ? $treeOptions.groupButtons : $treeOptions.nodeButtons;

  $: hasDropMenu = branch
    ? $treeOptions.groupMenuItems?.length
    : $treeOptions.nodeMenuDropItems?.length;

  $: dropMenuItems = branch
    ? $treeOptions.groupMenuDropItems
    : $treeOptions.nodeMenuDropItems;

  $: selectable = branch
    ? $treeOptions.groupSelectable
    : $treeOptions.nodeSelection;

  const handleClick = (e) => {
    if (disabled) return;

    if (children?.length || hasSlot || accordion) {
      open = !open;
      return;
    }

    if (selectable) {
      dispatch("nodeSelect", { id, label });
      return;
    }

    dispatch("nodeClick", { id, label });
  };

  const handleSelect = (e) => {
    if (selectable) {
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
  class:is-menu-open={openMenu}
  class:list={list || accordion}
>
  <a
    class="spectrum-TreeView-itemLink"
    class:list={list || accordion}
    class:branch={type == "branch"}
    style:padding-left={(children?.length && !accordion) ||
    hasButtons ||
    hasDropMenu
      ? "0.25rem"
      : flat || (accordion && !hasButtons)
        ? "0.65rem"
        : hasButtons && !accordion
          ? "1.25rem"
          : "1.5rem"}
    on:mouseenter={() => (hover = true)}
    on:mouseleave={() => (hover = false)}
    on:click|self|stopPropagation={accordion ? handleClick : handleSelect}
  >
    <div
      class="left-contents"
      class:has-buttons={hasButtons}
      on:mousedown={() => ($menuStore = id)}
    >
      {#if children?.length && !accordion}
        <i
          class="ri-arrow-right-s-line chevron"
          class:open
          class:childless={children?.length < 1}
          on:click|self|stopPropagation={handleClick}
        >
        </i>
      {/if}

      {#if hasDropMenu && accordion}
        <SuperButton
          bind:anchor={menuAnchor}
          size="S"
          icon={$treeOptions.nodeMenuIcon}
          fillOnHover
          quiet
          selected={$menuStore == id}
          on:click={(e) => {
            openMenu = !openMenu;
            $menuStore = openMenu ? id : false;
          }}
          text=""
        />
      {/if}

      {#if $treeOptions?.checkboxes && selectable}
        {#if selected}
          <i
            on:click|preventDefault|stopPropagation={handleSelect}
            class="ri-checkbox-fill checkbox"
          />
        {:else}
          <i
            on:click|preventDefault|stopPropagation={handleSelect}
            class="ri-checkbox-blank-line checkbox"
          />
        {/if}
      {/if}
      {#if hasButtons && !disabled}
        {#each buttons as { text, icon, disabled, quiet, onClick }}
          <SuperButton
            size="S"
            {icon}
            {text}
            {quiet}
            {disabled}
            fillOnHover={true}
            onClick={enrichButtonActions(onClick, $allContext)}
          />
        {/each}
      {/if}
      {#if icon}
        <i class={icon} class:icon />
      {/if}
      {label || "Not Set"}
    </div>

    <!-- The Node Action Menu  -->
    {#if hasDropMenu && (hover || openMenu) && !disabled && !accordion}
      <SuperButton
        bind:anchor={menuAnchor}
        size="S"
        icon={$treeOptions.nodeMenuIcon}
        fillOnHover
        quiet
        selected={$menuStore == id}
        on:click={(e) => {
          openMenu = !openMenu;
          $menuStore = openMenu ? id : false;
        }}
        text=""
      />
    {/if}

    {#if children?.length && accordion}
      <i
        class="ri-arrow-right-s-line chevron"
        class:open
        class:childless={children?.length < 1}
        on:click|self|stopPropagation={handleClick}
      >
      </i>
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
          type={node.type}
          icon={node.icon}
          {list}
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
{#if hasDropMenu && openMenu}
  <SuperPopover
    open={openMenu}
    align={"left"}
    anchor={menuAnchor}
    on:close={() => {
      openMenu = false;
      $menuStore = undefined;
    }}
  >
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div
      class="actionMenu"
      on:click={(e) => {
        openMenu = false;
      }}
    >
      {#each dropMenuItems as { text, icon, disabled, onClick }}
        <SuperButton
          size="M"
          {icon}
          {text}
          quiet
          {disabled}
          onClick={enrichButtonActions(onClick, $allContext)}
          on:click={() => ($menuStore = undefined)}
          menuItem
          menuAlign="left"
        />
      {/each}
    </div>
  </SuperPopover>
{/if}

<style>
  .spectrum-TreeView-item {
    transition: all 130ms;
    text-decoration: none !important;
  }
  .spectrum-TreeView-item.is-selected {
    background-color: var(--spectrum-global-color-gray-100) !important;
  }
  .spectrum-TreeView-itemLink:hover {
    transition: all 130ms;
    background-color: var(--spectrum-global-color-gray-75) !important;
  }
  .spectrum-TreeView-itemLink.branch {
    transition: all 130ms;
    font-weight: 600;
  }
  .is-menu-open {
    background-color: var(--spectrum-global-color-gray-100);
    transition: all 130ms;
  }
  .spectrum-TreeView-item.list {
    transition: all 130ms;
    border-bottom: 1px solid var(--spectrum-global-color-gray-200);
  }
  .spectrum-TreeView-itemLink {
    width: 100%;
    display: flex;
    justify-content: flex-start;
    gap: 0.25rem;
  }
  .spectrum-TreeView-itemLink.list {
    width: 100%;
    display: flex;
    justify-content: space-between;
    padding-right: 0.25rem;
  }
  .icon {
    font-size: 14px;
    color: var(--spectrum-global-color-gray-600);
    z-index: 1;
  }
  .checkbox {
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
    min-width: 120px;
    display: flex;
    flex-direction: column;
    align-items: stretch;
  }

  .left-contents {
    display: flex;
    align-items: center;
    gap: 0.25rem;
  }
  .left-contents.has-buttons {
    gap: 0rem;
  }
</style>
