<script>
  import { getContext, createEventDispatcher } from "svelte";
  import SuperPopover from "../../bb_super_components_shared/src/lib/SuperPopover/SuperPopover.svelte";
  import SuperButton from "../../bb_super_components_shared/src/lib/SuperButton/SuperButton.svelte";
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
  export let accordion;
  export let color;
  export let bgColor;
  export let iconColor;
  export let flat;
  export let row;
  export let group;

  let openMenu;
  let menuAnchor;

  $: context = {
    ...row,
    id,
    label,
    children,
  };

  $: if (disabled) open = false;
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
  $: selected = $selectionStore.findIndex((x) => x.id == id) > -1;

  $: hasChildren = children?.length;
  $: hasButtons = buttons?.length;

  $: buttons = groupBranch
    ? $treeOptions.groupButtons
    : $treeOptions.nodeButtons;

  $: hasDropMenu = groupBranch
    ? $treeOptions.groupMenuDropItems?.length
    : $treeOptions.nodeMenuDropItems?.length;

  $: dropMenuItems = groupBranch
    ? $treeOptions.groupMenuDropItems
    : $treeOptions.nodeMenuDropItems;

  $: hasCheboxes = $treeOptions.checkboxes && selectable;

  const handleClick = (e) => {
    if (disabled) return;

    if (children?.length || renderSlot || accordion) {
      open = !open;
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
<div
  class="tree-node"
  class:is-disabled={disabled}
  class:is-open={open}
  class:accordion
  class:groupBranch
  style:background-color={bgColor}
>
  <div
    class="tree-node-item"
    class:hasChildren={hasChildren || renderSlot}
    class:hasDropMenu={hasDropMenu && rightChevron && !hasCheboxes}
    class:hasButtons
    class:hasCheckbox={hasCheboxes}
    class:rightChevron
    class:accordion
    style:color
    class:selected
    class:disabled
    class:is-menu-open={openMenu}
    style:background-color={openMenu && !selected
      ? "var(--spectrum-global-color-gray-100)"
      : bgColor}
    on:click|self={handleClick}
  >
    {#if !rightChevron}
      <div
        class="control-content"
        class:flat={flat && !hasChildren && !renderSlot}
      >
        {#if (hasChildren || renderSlot) && $treeOptions.chevronPosition == "left"}
          <i
            class="ri-arrow-right-s-line chevron"
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
        <i class={icon} class:icon style:color={iconColor} />
      {/if}
      {#if hasDropMenu && rightChevron}
        <SuperButton
          size="XS"
          icon={$treeOptions.nodeMenuIcon ?? "ri-more-2-fill"}
          fillOnHover
          quiet
          selected={$menuStore == id}
          on:click={(e) => {
            menuAnchor = e.detail;
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
        {label || "Not Set"}
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
        class="ri-arrow-right-s-line chevron accordion"
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
      transition:slide={{ duration: 130 }}
    >
      {#each children as node}
        <svelte:self
          id={node.id}
          {hasSlot}
          renderSlot={node.renderSlot}
          label={node.label}
          children={node.children}
          quiet={node.quiet}
          open={node.open}
          type={node.type}
          icon={node.icon}
          color={node.color}
          bgColor={node.bgColor}
          group={node.group}
          {list}
          {accordion}
          row={node.row}
          on:nodeSelect
          on:nodeClick
          on:nodeAction
        >
          <slot />
        </svelte:self>
      {/each}

      {#if renderSlot && $treeOptions.hasSlot && (open || children.length == 0)}
        <Provider data={context} scope={ContextScopes.Local}>
          <slot />
        </Provider>
      {/if}
    </div>
  {/if}
</div>

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
      {#each dropMenuItems as { text, icon, disabled, onClick }}
        <SuperButton
          size="M"
          {icon}
          {text}
          quiet
          {disabled}
          on:click={() => {
            dispatch("nodeAction", {
              onClick,
              row,
              group: groupBranch ? label : group,
            });
            $menuStore = undefined;
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

  .chevron.accordion {
    color: var(--spectrum-global-color-gray-600);
    font-size: 18px;
  }
  .chevron.open {
    transform: rotate(90deg);
    color: var(--spectrum-global-color-gray-800);
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
    min-width: 1.4rem;
    display: flex;
    justify-content: center;

    &.flat {
      min-width: 0.75rem;
    }

    & > .chevron {
      transition: all 130ms;
      font-size: 16px;
      color: var(--spectrum-global-color-gray-600);

      &.open {
        color: var(--spectrum-global-color-gray-800);
      }
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
</style>
