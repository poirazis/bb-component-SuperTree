<script>
  import { getContext, setContext } from "svelte";
  import { writable } from "svelte/store";
  import CellString from "../../bb_super_components_shared/src/lib/SuperTableCells/CellString.svelte";

  export let menuItems;
  export let headerText;
  export let searchable;
  export let actionMenu;
  let menuAnchor;
  let hover, inEdit;
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
  class="searchHeader"
  style:border-bottom={inEdit
    ? "1px solid var(--spectrum-global-color-blue-400)"
    : "2px solid transparent"}
>
  {#if (searchable && hover && !disabled) || searchFilter?.length || inEdit}
    <div
      on:mouseleave={() => (hover = false)}
      style:display={"flex"}
      style:justify-content={"stretch"}
      style:flex={"auto"}
    >
      <CellString
        {cellOptions}
        on:change={handleSearch}
        on:enteredit={() => (inEdit = true)}
        on:exitedit={() => (inEdit = searchFilter?.length)}
      />
    </div>
  {:else}
    <div
      class="title"
      on:mouseenter={() => (hover = searchable)}
      on:mouseleave={() => (hover = false)}
    >
      {#if searchable}
        <i
          class="ri-search-line"
          style="font-size: 12px; color: var(--spectrum-global-color-gray-500)"
        />
      {/if}
      {headerText}
    </div>
  {/if}

  {#if actionMenu && menuItems?.length}
    {#if menuItems.length > 1}
      <button
        bind:this={menuAnchor}
        class="spectrum-ActionButton spectrum-ActionButton--sizeM spectrum-ActionButton--quiet"
        class:is-selected={openMenu}
        on:click={handleMenu}
      >
        <i class={menuIcon} />
      </button>
    {:else}
      <Block>
        {#each menuItems as { text, icon, disabled, onClick, quiet }}
          <BlockComponent
            type="plugin/bb-component-SuperButton"
            props={{
              size: "M",
              icon,
              text,
              quiet,
              disabled,
              onClick,
            }}
          ></BlockComponent>
        {/each}
      </Block>
    {/if}
  {/if}
</div>

<style>
  .spectrum-TreeView {
    flex: 1 1 auto;
    margin: unset;
    overflow-y: auto;
  }

  .searchHeader {
    min-width: 200px;
    width: 100%;
    height: 2rem;
    display: flex;
    align-items: center;
    justify-content: space-between;
    transition: all 130ms;
    margin-bottom: 0.25rem;
    border: 1px solid white;
  }

  .actionMenu {
    width: 100%;
    display: flex;
    flex-direction: column;
    align-items: stretch;
  }

  .title {
    font-size: 13px;
    font-weight: 700;
    letter-spacing: 0.72px;
    text-transform: uppercase;
    color: var(--spectrum-global-color-gray-700);
    transition: all 230ms;
    padding-left: 0.25rem;
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow-x: hidden;
    display: flex;
    align-items: center;
    gap: 0.65rem;
    padding-left: 0.25rem;
  }
</style>
