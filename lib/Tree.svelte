<script>
  import { getContext, createEventDispatcher } from "svelte";
  import { SuperButton, SuperPopover } from "@poirazis/supercomponents-shared";
  const { Provider, ContextScopes } = getContext("sdk");
  const treeOptions = getContext("superTreeOptions");
  import { slide } from "svelte/transition";

  const dispatch = createEventDispatcher();

  export let label;
  export let children = [];
  export let open = false;
  export let hasSlot;
  export let renderSlot;
  export let id;
  export let disabled;
  export let list;
  export let type;
  export let icon;
  export let color;
  export let bgColor;
  export let iconColor;
  export let flat;
  export let row;
  export let group;
  export let visible;

  let openMenu;
  let menuAnchor;

  $: context = {
    ...row,
    id,
    label,
    children,
    group: groupBranch ? label : group,
  };

  $: isRoot = id == "tree-root";
  $: if (disabled) open = false;
  $: showCount = $treeOptions.showCount;
  $: groupBranch = type == "groupBranch";
  $: rightChevron = $treeOptions.chevronPosition == "right";
  $: selectable =
    id == "tree-root" || disabled
      ? false
      : groupBranch
        ? $treeOptions.groupSelectable
        : $treeOptions.nodeSelection;

  $: selectionStore = $treeOptions.selectedNodes;
  $: menuStore = $treeOptions.menuStore;
  $: selected = $selectionStore.findIndex((x) => x == id) > -1;

  $: hasChildren = children?.length;
  $: hasButtons = !isRoot && buttons?.length;

  $: buttons = groupBranch
    ? $treeOptions.groupButtons
    : isRoot
      ? $treeOptions.rootButtons
      : $treeOptions.nodeButtons;

  $: hasDropMenu = groupBranch
    ? $treeOptions.groupMenuDropItems?.length
    : isRoot
      ? $treeOptions.rootButtons?.length
      : $treeOptions.nodeMenuDropItems?.length;

  $: dropMenuItems = groupBranch
    ? $treeOptions.groupMenuDropItems
    : isRoot
      ? $treeOptions.rootButtons
      : $treeOptions.nodeMenuDropItems;

  $: hasCheboxes = $treeOptions.checkboxes && selectable;

  function hasSelectedDescendant(children) {
    if (!children) return false;
    for (let child of children) {
      if ($selectionStore.includes(child.id)) return true;
      if (child.children && hasSelectedDescendant(child.children)) return true;
    }
    return false;
  }

  const handleClick = (e) => {
    if (disabled) return;

    if (children?.length || renderSlot) {
      // Prevent collapsing if any descendant is selected
      if (!hasSelectedDescendant(children)) {
        open = !open;
      }
      return;
    }

    if (selectable) {
      dispatch("nodeSelect", { id, label, row });
      return;
    }

    dispatch("nodeClick", { id, label, row });
  };

  const handleSelect = (e) => {
    if (selectable) {
      dispatch("nodeSelect", { id, label, row });
      return;
    }
  };
</script>

<!-- svelte-ignore a11y-missing-attribute -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
{#if visible !== false}
  <button
    class="tree-node"
    class:is-disabled={disabled}
    class:is-open={open}
    style:background-color={bgColor}
    on:contextmenu|preventDefault|stopPropagation={(e) => {
      menuAnchor = e.target;
      openMenu = !openMenu;
      $menuStore = openMenu ? id : false;
    }}
  >
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div
      class="tree-node-item"
      class:hasChildren={hasChildren || renderSlot}
      class:hasDropMenu={hasDropMenu && rightChevron && !hasCheboxes}
      class:hasButtons
      class:hasCheckbox={hasCheboxes}
      class:rightChevron
      style:color
      class:selected
      class:disabled
      class:is-menu-open={openMenu}
      on:click|self={handleClick}
    >
      {#if !rightChevron}
        <div
          class="control-content"
          class:flat={flat && !hasChildren && !renderSlot && !isRoot}
        >
          {#if (hasChildren || renderSlot) && $treeOptions.chevronPosition == "left"}
            <i
              class="ph ph-caret-right chevron"
              class:locked={hasSelectedDescendant(children, $selectionStore)}
              class:open
              on:click={handleClick}
            >
            </i>
          {/if}
        </div>
      {/if}

      <div class="left-contents" class:hasButtons>
        {#if hasCheboxes}
          {#if selected}
            <i
              on:click|preventDefault|stopPropagation={handleSelect}
              class="ph ph-check-square"
            />
          {:else}
            <i
              on:click|preventDefault|stopPropagation={handleSelect}
              class="ph ph-square"
            />
          {/if}
        {/if}
        {#if hasButtons && !disabled}
          {#each buttons as { text, icon, disabled, quiet, onClick, type }}
            <SuperButton
              size="S"
              {icon}
              {text}
              {quiet}
              {disabled}
              fillOnHover={true}
              {type}
              on:click={() => {
                dispatch("nodeAction", {
                  onClick,
                  row,
                  group: groupBranch ? label : group,
                });
              }}
            />
          {/each}
        {/if}
        {#if icon}
          <i class={"ph ph-" + icon} class:icon style:color={iconColor} />
        {/if}
        {#if hasDropMenu && rightChevron}
          <SuperButton
            size="XS"
            icon={$treeOptions.nodeMenuIcon ?? "ri-more-2-fill"}
            fillOnHover
            quiet
            selected={$menuStore == id}
            onClick={(e) => {
              menuAnchor = e.target;
              openMenu = !openMenu;
              $menuStore = openMenu ? id : false;
            }}
            text=""
          />
        {/if}
        <div
          class="label"
          class:selectable
          class:branch={type == "linkBranch"}
          style="z-index: 1;"
          on:mousedown={selectable ? handleSelect : handleClick}
        >
          {@html label ?? "Not Set"}
          {#if showCount && (isRoot || hasChildren)}
            <span class="children-count">
              ({children?.filter((n) => n.visible)?.length})
            </span>
          {/if}
        </div>
      </div>

      <!-- The Node Action Menu  -->
      {#if hasDropMenu && !rightChevron}
        <i
          class="ri-more-fill menu-icon"
          on:click={(e) => {
            menuAnchor = e.target;
            openMenu = !openMenu;
            $menuStore = openMenu ? id : false;
          }}
        />
      {/if}

      {#if hasChildren && $treeOptions.chevronPosition == "right"}
        <i
          class="ri-arrow-right-s-line chevron"
          on:mousedown={handleClick}
          class:open
        >
        </i>
      {/if}
    </div>

    {#if (children?.length || renderSlot) && open}
      <div
        class="tree"
        style:display={open ? "flex" : "none"}
        transition:slide={{ duration: 230 }}
      >
        {#each children as node}
          <svelte:self
            {...node}
            {hasSlot}
            {list}
            {flat}
            {disabled}
            on:nodeSelect
            on:nodeClick
            on:nodeAction
          >
            <slot />
          </svelte:self>
        {/each}

        {#if renderSlot && $treeOptions.hasSlot}
          <Provider data={context} scope={ContextScopes.Local}>
            <slot />
          </Provider>
        {/if}
      </div>
    {/if}
  </button>
{/if}

{#if hasDropMenu && openMenu}
  <SuperPopover
    open
    align={"left"}
    anchor={menuAnchor}
    on:close={() => {
      openMenu = false;
      $menuStore = undefined;
    }}
  >
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div class="action-menu">
      {#each dropMenuItems as { text, icon, disabled, onClick, type }}
        <SuperButton
          size="M"
          {icon}
          {text}
          {type}
          quiet
          {disabled}
          onClick={() => {
            dispatch("nodeAction", {
              onClick,
              id,
              label,
              row,
              group: groupBranch ? label : group,
            });
            setTimeout(() => {
              $menuStore = undefined;
            }, 100);
            openMenu = false;
          }}
          menuItem
          menuAlign="left"
        />
      {/each}
    </div>
  </SuperPopover>
{/if}

<style>
  .branch {
    font-weight: 600;
    color: var(--spectrum-global-color-gray-600) !important;
  }
  .icon {
    font-size: 14px;
    color: var(--spectrum-global-color-gray-600);
    margin-right: 4px;
  }
  .checkbox {
    font-size: 14px;
    color: var(--spectrum-global-color-gray-600);
  }

  .chevron {
    transition: all 130ms;
    color: var(--spectrum-global-color-gray-700);
  }
  .chevron.open {
    transform: rotate(90deg);
    color: var(--spectrum-global-color-gray-800);
  }

  .chevron.locked {
    opacity: 0.5;
    pointer-events: none;
  }

  .children-count {
    color: var(--spectrum-global-color-gray-600);
    margin-left: 0.25rem;
    font-size: smaller;
    font-family: monospace;
  }

  .action-menu {
    min-width: 160px;
    padding: 0.25rem;
    display: flex;
    flex-direction: column;
    align-items: stretch;
  }

  .left-contents {
    display: flex;
    align-items: center;
    gap: 0.25rem;
    padding-right: 0.5rem;
    min-width: 0;
  }

  .left-contents.hasButtons {
    gap: 0.5rem;
  }

  .control-content {
    min-width: 1.75rem;
    display: flex;
    padding-left: 0.25rem;
    align-items: center;
    justify-content: center;

    &.flat {
      min-width: 1rem;
    }
  }

  .left-contents > .label {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  .label.selectable:hover {
    cursor: pointer;
  }

  .highlight {
    background-color: var(--spectrum-global-color-yellow-400);
    color: var(--spectrum-global-color-gray-900);
  }
</style>
