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
  $: linkBranch = type == "linkBranch";
  $: linkItem = type == "linkItem";

  $: selectionStore = $treeOptions.selectedNodes;
  $: menuStore = $treeOptions.menuStore;
  $: selected = $selectionStore.findIndex((x) => x.id == id) > -1;

  $: hasChildren = children?.length;
  $: hasCheboxes = $treeOptions.checkboxes || (accordion && selectable);

  $: hasButtons = branch
    ? $treeOptions?.groupButtons?.length
    : $treeOptions?.nodeButtons?.length;

  $: buttons = branch
    ? $treeOptions.groupButtons
    : $treeOptions.nodeMenu
      ? $treeOptions.nodeButtons
      : false;

  $: hasDropMenu = branch
    ? $treeOptions.groupMenuItems?.length
    : linkItem
      ? $treeOptions.linkMenuItems?.length
      : linkBranch
        ? false
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
      e.stopPropagation();
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
    style:padding-left={accordion
      ? hasButtons || hasCheboxes
        ? "0rem"
        : "0.5rem"
      : hasChildren
        ? "0.25rem"
        : hasButtons
          ? "1.25rem"
          : "1.5rem"}
    on:mouseenter={() => (hover = true)}
    on:mouseleave={() => (hover = false)}
    on:mousedown|self|preventDefault={handleClick}
  >
    <div class="left-contents" class:has-buttons={hasButtons}>
      {#if hasChildren && !accordion}
        <i
          class="ri-arrow-right-s-line chevron"
          class:open
          on:click={handleClick}
        >
        </i>
      {/if}

      {#if hasDropMenu && accordion}
        <SuperButton
          bind:anchor={menuAnchor}
          size="XS"
          icon={$treeOptions.nodeMenuIcon ?? "ri-more-2-fill"}
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
      {#if hasCheboxes}
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
      <div
        class="label"
        class:selectable
        class:branch={type == "linkBranch"}
        style="z-index: 1;"
        on:mousedown={selectable ? handleSelect : handleClick}
      >
        {label || "Not Set"}
      </div>
    </div>

    <!-- The Node Action Menu  -->
    {#if hasDropMenu && (hover || openMenu) && !disabled && !accordion}
      <SuperButton
        bind:anchor={menuAnchor}
        size="XS"
        icon="ri-more-fill"
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

    {#if hasChildren && accordion}
      <i class="ri-arrow-right-s-line chevron accordion" class:open> </i>
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
          {accordion}
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
    border: unset;
  }
  .spectrum-TreeView-item.is-selected {
    background-color: var(--spectrum-global-color-gray-100) !important;
  }
  .is-menu-open {
    background-color: var(--spectrum-global-color-gray-100);
    transition: all 130ms;
  }
  .spectrum-TreeView-item.list {
    transition: all 130ms;
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
    padding-right: 0.35rem;
    border-bottom: 1px solid var(--spectrum-global-color-gray-200);
  }

  .spectrum-TreeView-itemLink:hover {
    transition: all 130ms;
    background-color: var(--spectrum-global-color-gray-75) !important;
  }
  .branch {
    transition: all 130ms;
    font-weight: 600;
    color: var(--spectrum-global-color-gray-600) !important;
  }
  .icon {
    font-size: 14px;
    color: var(--spectrum-global-color-gray-600);
  }
  .checkbox {
    font-size: 16px;
    color: var(--spectrum-global-color-gray-600);
    z-index: 1;
    margin-right: 4px;
  }
  .chevron {
    transition: all 130ms;
    font-size: 16px;
    color: var(--spectrum-global-color-gray-500);
  }

  .chevron.accordion {
    color: var(--spectrum-global-color-gray-600);
    font-size: 18px;
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
    padding-left: 0.25rem;
  }

  .label.selectable:hover {
    cursor: pointer;
  }
</style>
