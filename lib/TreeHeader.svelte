<script>
  import { getContext, createEventDispatcher } from "svelte";

  import CellString from "../../bb_super_components_shared/src/lib/SuperTableCells/CellString.svelte";
  import SuperPopover from "../../bb_super_components_shared/src/lib/SuperPopover/SuperPopover.svelte";
  import SuperButton from "../../bb_super_components_shared/src/lib/SuperButton/SuperButton.svelte";
  import { readonly } from "svelte/store";

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
    debounce: 200,
    clearValueIcon: true,
    role: "formInput",
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
    dispatch("search", e.detail);
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
        <SuperButton
          size="S"
          icon={headerMenuIcon}
          quiet
          selected={openMenu}
          on:click={handleMenu}
          text=""
        />
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
      {#each headerDropMenuItems as { text, icon, disabled, enrichedOnClick, quiet }}
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
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: space-between;
    transition: all 130ms;
    height: 2.4rem;
    padding-left: 0.25rem;
    padding-right: 0.5rem;
    color: var(--spectrum-global-color-gray-800);

    &:hover {
      color: var(--spectrum-global-color-gray-800);
    }

    & > i {
      &:hover {
        cursor: pointer;
        color: var(--primaryColor);
      }
    }
  }
  .searchHeader.list {
    background-color: var(--spectrum-global-color-gray-100);
  }
  .searchHeader.list.quiet {
    background-color: transparent !important;
    padding-left: none !important;
  }

  .searchHeader.quiet {
    background-color: transparent !important;
    padding-left: none;
  }
  .inEdit {
    background-color: var(--spectrum-global-color-gray-50) !important;
  }
  .actionMenu {
    min-width: 120px;
    display: flex;
    flex-direction: column;
    align-items: stretch;
  }

  .action-buttons {
    flex: none;
    display: flex;
    align-items: center;
    gap: 0.25rem;
  }

  .title {
    flex: auto;
    font-size: 12px;
    font-weight: 500;
    text-transform: uppercase;
    letter-spacing: 1.2px;
    transition: all 130ms;
    padding-left: 0.35rem;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow-x: hidden;
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
</style>
