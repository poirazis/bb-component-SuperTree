<script>
  import { getContext, createEventDispatcher, onDestroy } from "svelte";
  import { SuperButton, SuperPopover } from "@poirazis/supercomponents-shared";
  const { Provider, ContextScopes } = getContext("sdk");
  const treeOptions = getContext("superTreeOptions");
  import { slide } from "svelte/transition";

  const dispatch = createEventDispatcher();

  onDestroy(() => {
    if (tooltipTimer) {
      clearTimeout(tooltipTimer);
    }
  });

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
  export let isGroup;
  export let visible;

  let openMenu;
  let menuAnchor;
  let labelElement;
  let tooltipOpen = false;
  let tooltipAnchor;
  let tooltipTimer;

  $: context = {
    ...row,
    id,
    label,
    children,
    isGroup,
    group: groupBranch ? label : undefined,
  };

  $: isRoot = id == "tree-root";
  $: if (disabled) open = false;
  $: showCount = $treeOptions.showCount;
  $: groupBranch = isGroup;
  $: rightChevron = $treeOptions.chevronPosition == "right";
  $: selectable =
    id == "tree-root" || disabled
      ? false
      : groupBranch
        ? $treeOptions.groupSelectable
        : $treeOptions.nodeSelection;

  $: selectionStore = $treeOptions.selectedNodes;
  $: selectedGroupsStore = $treeOptions.selectedGroups;
  $: menuStore = $treeOptions.menuStore;
  $: selected =
    $selectionStore.findIndex((x) => x == id) > -1 ||
    (isGroup && $selectedGroupsStore.findIndex((x) => x == id) > -1);

  $: hasChildren = children?.length;

  $: dropMenuItems = groupBranch
    ? $treeOptions.groupMenuDropItems
    : isRoot
      ? $treeOptions.rootButtons
      : $treeOptions.nodeMenuDropItems;

  $: hasDropMenu = dropMenuItems && dropMenuItems.length > 0;
  $: hasCheboxes = $treeOptions.checkboxes && selectable;
  $: isTrimmed =
    labelElement && labelElement.scrollWidth > labelElement.clientWidth;

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
      dispatch("nodeSelect", { id, label, row, isGroup });
      return;
    }

    dispatch("nodeClick", { id, label, row, isGroup });
  };

  const handleSelect = (e) => {
    if (selectable) {
      dispatch("nodeSelect", {
        id,
        label,
        row,
        isGroup,
      });
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
      class:hasCheckbox={hasCheboxes}
      class:rightChevron
      style:color
      class:selected
      class:disabled
      class:selectable={selectable && !disabled}
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

      <div class="left-contents">
        {#if hasDropMenu && rightChevron}
          <i
            class="ph ph-dots-three-vertical menu-icon left"
            on:click={(e) => {
              menuAnchor = e.target;
              openMenu = !openMenu;
              $menuStore = openMenu ? id : false;
            }}
          />
        {/if}

        {#if hasCheboxes}
          <i
            class="ph"
            class:selected
            class:ph-check-square={selected}
            class:ph-square={!!selectable && !selected}
            on:click={handleSelect}
          ></i>
        {/if}

        {#if icon}
          <i
            class="ph ph-{icon} node-icon"
            class:selected
            style="color: {iconColor ||
              color ||
              'var(--spectrum-global-color-gray-700)'}"
          ></i>
        {/if}

        <div
          class="label"
          class:selectable
          class:branch={type == "linkBranch"}
          style="z-index: 1;"
          bind:this={labelElement}
          on:mousedown={selectable ? handleSelect : handleClick}
          on:mouseenter={() => {
            if (isTrimmed) {
              tooltipTimer = setTimeout(() => {
                tooltipOpen = true;
                tooltipAnchor = labelElement;
              }, 500);
            }
          }}
          on:mouseleave={() => {
            if (tooltipTimer) {
              clearTimeout(tooltipTimer);
              tooltipTimer = null;
            }
            tooltipOpen = false;
          }}
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
          class="ph ph-dots-three menu-icon"
          on:click={(e) => {
            menuAnchor = e.target;
            openMenu = !openMenu;
            $menuStore = openMenu ? id : false;
          }}
        />
      {/if}

      {#if hasChildren && rightChevron}
        <i
          class="ph ph-caret-right chevron"
          class:locked={hasSelectedDescendant(children, $selectionStore)}
          class:open
          on:click={handleClick}
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

{#if tooltipOpen}
  <SuperPopover
    open
    align="center"
    anchor={tooltipAnchor}
    on:close={() => (tooltipOpen = false)}
  >
    <div class="tooltip-content">{@html label}</div>
  </SuperPopover>
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

  .node-icon {
    font-size: 1rem;
    margin-right: 0.25rem;
    opacity: 0.85;
  }

  .node-icon.selected {
    opacity: 1;
  }

  .menu-icon {
    font-size: 1rem;
    color: var(--spectrum-global-color-gray-600);
    margin-right: 0.75rem;
    cursor: pointer;
  }

  .menu-icon.left {
    margin-right: 0.25rem;
    margin-left: 0.25;
  }

  .ph.ph-square {
    font-size: 1rem;
    font-weight: 300;
    color: var(--spectrum-global-color-gray-600);
  }
  .ph.ph-check-square {
    font-size: 1rem;
    font-weight: 500;
    color: var(--spectrum-global-color-gray-800);
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

  .tooltip-content {
    flex: auto;
    display: flex;
    align-items: center;
    max-width: 300px;
    word-wrap: break-word;
    white-space: normal;
    padding: 0.25rem 0.5rem;
    line-height: 12px;
    background-color: var(--spectrum-global-color-blue-100);
  }
</style>
