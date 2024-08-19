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
    role: "inlineInput",
  };

  $: enrichButtons(headerDropMenuItems, $context);
  const enrichButtons = (buttons) => {
    buttons?.forEach((btn) => {
      btn.enrichedOnClick = enrichButtonActions(btn.onClick, $context);
    });
  };

  const handleSearch = (e) => {
    searchFilter = e.detail;
    dispatch("search", e.detail);
  };

  const handleMenu = (e) => {
    openMenu = !openMenu;
  };
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
  class="searchHeader"
  class:list={!quiet}
  class:quiet
  class:inEdit
  on:mouseleave={() => (hover = false)}
>
  {#if (searchable && hover && !disabled) || searchFilter || inEdit}
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <CellString
      {cellOptions}
      value={searchFilter}
      on:change={handleSearch}
      on:enteredit={() => (inEdit = true)}
      on:exitedit={() => (inEdit = searchFilter?.length)}
    />
  {:else}
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div class="title" on:mouseenter={() => (hover = searchable)}>
      {headerText}
    </div>
  {/if}

  {#if (headerDropMenuItems?.length || headerButtons?.length) && !inEdit}
    <div class="action-buttons">
      {#each headerButtons as { text, icon, size, disabled, onClick, quiet }}
        <SuperButton
          {size}
          {icon}
          {disabled}
          {quiet}
          onClick={enrichButtonActions(onClick, $context)}
          {text}
        />
      {/each}

      {#if headerDropMenuItems?.length}
        <SuperButton
          bind:anchor={menuAnchor}
          size="XS"
          icon={headerMenuIcon}
          quiet
          selected={openMenu}
          onClick={handleMenu}
          text=""
        />
        <SuperPopover
          open={openMenu}
          anchor={menuAnchor}
          on:close={() => (openMenu = false)}
        >
          <!-- svelte-ignore a11y-click-events-have-key-events -->
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
    </div>
  {/if}
</div>

<style>
  .searchHeader {
    flex: none;
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: space-between;
    transition: all 130ms;
    border: 4px solid transparent;
    height: 2.2rem;
    border-top-right-radius: 4px;
    border-top-left-radius: 4px;
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
    border-color: var(--spectrum-global-color-gray-200);
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
  }

  .title {
    flex: auto;
    font-size: 12px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 1.1px;
    color: var(--spectrum-global-color-gray-700);
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
