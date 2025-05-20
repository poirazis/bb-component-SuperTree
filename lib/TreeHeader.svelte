<script>
  import { getContext, createEventDispatcher } from "svelte";

  import CellString from "../../bb_super_components_shared/src/lib/SuperTableCells/CellString.svelte";
  import SuperPopover from "../../bb_super_components_shared/src/lib/SuperPopover/SuperPopover.svelte";
  import SuperButton from "../../bb_super_components_shared/src/lib/SuperButton/SuperButton.svelte";

  const { enrichButtonActions } = getContext("sdk");
  const context = getContext("context");

  const dispatch = createEventDispatcher();

  export let quiet;
  export let searchable;
  export let disabled;
  export let headerText;
  export let headerDropMenuItems;
  export let headerButtons;
  export let headerMenuIcon;
  export let inBuilder;

  let inEdit;
  let hover;
  let menuAnchor;
  let openMenu;
  let searchFilter;

  $: cellOptions = {
    placeholder: "Search...",
    icon: "ri-search-line",
    disabled,
    padding: "0.5rem",
    debounce: 500,
    clearValueIcon: true,
    role: "inlineInput",
    readonly: inBuilder,
  };

  $: enrichButtons(headerDropMenuItems, $context);

  const enrichButtons = (buttons) => {
    buttons?.forEach((btn) => {
      btn.enrichedOnClick = enrichButtonActions(btn.onClick, $context);
    });
  };

  const handleSearch = (e) => {
    if (inBuilder) return;
    searchFilter = e.detail;
    dispatch("search", searchFilter);
  };

  const handleMenu = (e) => {
    openMenu = !openMenu;
  };
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
  bind:this={menuAnchor}
  class="searchHeader"
  class:list={!quiet}
  class:quiet
  class:inEdit
  class:searchable
  class:filtered={searchFilter}
  on:mouseleave={() => (hover = false)}
>
  {#if (searchable && hover && !disabled) || searchFilter || inEdit}
    <CellString
      {cellOptions}
      value={searchFilter}
      on:change={handleSearch}
      on:enteredit={() => (inEdit = true)}
      on:exitedit={() => (inEdit = searchFilter?.length)}
    />
  {:else}
    <div class="title" on:mouseenter={() => (hover = searchable)}>
      {headerText}
    </div>
  {/if}

  {#if (headerDropMenuItems?.length || headerButtons?.length) && !inEdit}
    <div class="action-buttons">
      {#each headerButtons as { text, icon, size, disabled, onClick, quiet, type }}
        <SuperButton
          {size}
          {icon}
          {disabled}
          {quiet}
          onClick={enrichButtonActions(onClick, $context)}
          {text}
          {type}
        />
      {/each}

      {#if headerDropMenuItems?.length}
        <!-- svelte-ignore a11y-click-events-have-key-events -->
        <span class="drop-menu-icon" on:click={handleMenu}>
          <i class={headerMenuIcon} />
        </span>
      {/if}
    </div>
  {/if}
</div>

{#if headerDropMenuItems?.length}
  <SuperPopover
    open={openMenu}
    anchor={menuAnchor}
    on:close={() => (openMenu = false)}
  >
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div
      class="actionMenu"
      on:click|preventDefault={(e) => {
        openMenu = false;
      }}
    >
      {#each headerDropMenuItems as { text, icon, disabled, enrichedOnClick }}
        <SuperButton
          size="M"
          {icon}
          {disabled}
          quiet
          menuItem={true}
          menuAlign="right"
          onClick={enrichedOnClick}
          {text}
        />
      {/each}
    </div>
  </SuperPopover>
{/if}

<style>
  .searchHeader {
    flex: none;
    width: 100%;
    display: flex;
    align-items: stretch;
    justify-content: space-between;
    height: 38px;
    color: var(--spectrum-global-color-gray-800);
    border-bottom: 1px solid transparent;

    &.quiet {
      background-color: unset;
      border-bottom: none;
      color: var(--spectrum-global-color-gray-700);
    }

    & > i {
      &:hover {
        cursor: pointer;
        color: var(--primaryColor);
      }
    }

    &.inEdit {
      border-bottom: 1px solid var(--spectrum-global-color-gray-400);
    }

    &.filtered {
      border-bottom: 1px solid var(--spectrum-global-color-blue-400);
    }

    & > .title {
      align-self: center;
      font-size: 12px;
      font-weight: 500;
      text-transform: uppercase;
      letter-spacing: 1.2px;
      white-space: nowrap;
      text-overflow: ellipsis;
      overflow: hidden;
      padding-left: 0.65rem;
    }
  }
  .actionMenu {
    min-width: 160px;
    display: flex;
    flex-direction: column;
    align-items: stretch;
    padding: 0.25rem;
  }

  .action-buttons {
    flex: none;
    display: flex;
    align-items: center;
    gap: 0.25rem;
    & > .drop-menu-icon {
      min-width: 1.25rem;
      aspect-ratio: 1;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 12px;
      margin-right: 0.25rem;
      margin-left: 0.25rem;

      &:hover {
        color: var(--spectrum-global-color-gray-900);
        cursor: pointer;
      }
    }
  }
</style>
