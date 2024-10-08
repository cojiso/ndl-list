<!-- src/routes/NDCListColumn.svelte -->

<script lang="ts">
  import { onMount, createEventDispatcher } from 'svelte';
  import { ndc9Store } from '../store/ndc9Store';
  import type { NDC9Item } from '../lib/ndc9';

  const dispatch = createEventDispatcher();

  export let selectedItem: NDC9Item | null = null;
  export let scrollToSelected = false;

  let ndc9Items: NDC9Item[] = [];
  let stickyItems: NDC9Item[] = [];
  let containerElement: HTMLElement;
  let stickyHeader: HTMLElement;
  let lastUpdateTime = 0;
  let lastStickyItems: string[] = [];
  let updateQueued = false;
  const DEBOUNCE_THRESHOLD = 100;
  let ndcSearchQuery = '';
  let depthFilter = Infinity;

  ndc9Store.subscribe(value => {
    ndc9Items = value;
  });

  onMount(() => {
    ndc9Store.loadData();
  });

  $: if (selectedItem && scrollToSelected) {
    console.log('NDCListColumn: Scrolling to selected item', selectedItem);
    scrollToItem(selectedItem);
    scrollToSelected = false;
  }

  function scrollToItem(item: NDC9Item) {
    console.log('NDCListColumn: Attempting to scroll to item', item);
    const element = document.getElementById(item['@id']);
    if (element) {
      console.log('NDCListColumn: Element found, scrolling');
      element.scrollIntoView({ behavior: 'smooth', block: 'center' });
    } else {
      console.log('NDCListColumn: Element not found');
    }
  }

  function handleNdcSearch() {
    if (ndcSearchQuery) {
      const matchedItem = ndc9Items.find(item => item['skos:notation'] === ndcSearchQuery);
      if (matchedItem) {
        scrollToItem(matchedItem);
      } else {
        const closestItem = ndc9Items.reduce((closest, current) => {
          if (current['skos:notation'] && current['skos:notation'].startsWith(ndcSearchQuery)) {
            if (!closest || current['skos:notation'].length < closest['skos:notation'].length) {
              return current;
            }
          }
          return closest;
        }, null as NDC9Item | null);

        if (closestItem) {
          scrollToItem(closestItem);
        } else {
          console.log('No matching or close items found');
        }
      }
    }
  }

  function handleScroll() {
    if (!updateQueued) {
      updateQueued = true;
      requestAnimationFrame(updateStickyHeader);
    }
  }

  function updateStickyHeader() {
    updateQueued = false;
    const currentTime = Date.now();
    
    if (currentTime - lastUpdateTime < DEBOUNCE_THRESHOLD) {
      requestAnimationFrame(updateStickyHeader);
      return;
    }

    if (!stickyHeader || !containerElement) return;

    const stickyBottom = stickyHeader.getBoundingClientRect().bottom;
    const visibleIndex = binarySearch(stickyBottom);
    if (visibleIndex === -1) return;

    const visibleItem = ndc9Items[visibleIndex];
    const newStickyItems = visibleItem.ancestors;

    const newStickyItemIds = newStickyItems.map(id => id);
    if (!areArraysEqual(newStickyItemIds, lastStickyItems)) {
      stickyItems = newStickyItems
        .map(id => ndc9Items.find(item => item['@id'] === id))
        .filter((item): item is NDC9Item => item !== undefined)
        .sort((a, b) => a.depth - b.depth);

      lastStickyItems = newStickyItemIds;
      lastUpdateTime = currentTime;
    }
  }

  function binarySearch(targetPosition: number): number {
    let low = 0;
    let high = ndc9Items.length - 1;

    while (low <= high) {
      const mid = Math.floor((low + high) / 2);
      const element = document.getElementById(ndc9Items[mid]['@id']);
      if (!element) return -1;

      const rect = element.getBoundingClientRect();
      const itemCenter = (rect.top + rect.bottom) / 2;

      if (itemCenter <= targetPosition && rect.bottom > targetPosition) {
        return mid;
      } else if (itemCenter > targetPosition) {
        high = mid - 1;
      } else {
        low = mid + 1;
      }
    }

    return low > 0 ? low - 1 : 0;
  }

  function areArraysEqual(arr1: string[], arr2: string[]): boolean {
    if (arr1.length !== arr2.length) return false;
    return arr1.every((value, index) => value === arr2[index]);
  }

  function getIndentStyle(depth: number): string {
    return `padding-left: ${depth * 20}px;`;
  }

  function handleItemClick(item: NDC9Item) {
    console.log('ndccolumn: dispatch Item selected', item['@id']);
    dispatch('selectItem', { item, scrollTo: false });
  }

  $: filteredItems = ndc9Items.filter(item => 
    item['skos:notation']?.includes(ndcSearchQuery) && item.depth <= depthFilter
  );

  function handleDepthFilterChange(event: Event) {
    const selectedValue = (event.target as HTMLSelectElement).value;
    depthFilter = selectedValue === 'infinity' ? Infinity : parseInt(selectedValue, 10);
  }
</script>

<div class="ndc-column">
  <div class="ndc-search">
    <div class="depth-filter">
      <select on:change={handleDepthFilterChange}>
        <option value="infinity">∞</option>
        <option value="0">1<span class="unit">次</span></option>
        <option value="1">2<span class="unit">次</span></option>
        <option value="2">3<span class="unit">次</span></option>
        <option value="3">4<span class="unit">次</span></option>
        <option value="4">5<span class="unit">次</span></option>
      </select>
    </div>
    <input 
      type="text" 
      bind:value={ndcSearchQuery} 
      placeholder="NDC番号検索"
      on:keydown={(e) => e.key === 'Enter' && handleNdcSearch()}
    >
    <button on:click={handleNdcSearch}>Jump</button>
  </div>
  <div class="ndc-container" bind:this={containerElement} on:scroll={handleScroll}>
    <div class="sticky-header" bind:this={stickyHeader}>
      {#each stickyItems as item (item['@id'])}
        <div 
          class="sticky-item"
          style={getIndentStyle(item.depth)}
        >
          {#if item['skos:notation']}
            <span class="item-notation">{item['skos:notation']}</span>
          {/if}
          <span class="item-label">{item.prefLabel}</span>
        </div>
      {/each}
    </div>
    <div class="ndc-list">
      {#each filteredItems as item (item['@id'])}
      <div 
        class="ndc-item"
        style={getIndentStyle(item.depth)}
        id={item['@id']}
        on:click={() => handleItemClick(item)}
        on:keydown={(e) => e.key === 'Enter' && handleItemClick(item)}
        tabindex="0"
        role="button"
      >
        {#if item['skos:notation']}
          <span class="item-notation">{item['skos:notation']}</span>
        {/if}
        <span class="item-label">{item.prefLabel}</span>
      </div>
      {/each}
    </div>
  </div>
</div>

<style>
  .ndc-column {
    display: flex;
    flex-direction: column;
    height: 100%;
  }

  .ndc-search {
    margin-bottom: var(--spacing-small);
    display: flex;
    gap: 6px;
  }

  .ndc-search input {
    flex-grow: 1;
    padding: var(--spacing-small);
    font-size: var(--font-size-base);
    border: 1px solid var(--color-secondary);
    border-radius: 4px;
  }

  .ndc-search button {
    padding: var(--spacing-small) 12px;
    font-size: var(--font-size-base);
    cursor: pointer;
    background-color: var(--color-primary);
    color: var(--color-background);
    border: none;
    border-radius: 4px;
  }

  .ndc-search button:hover {
    background-color: var(--color-primary-dark);
  }

  .ndc-container {
    flex: 1;
    overflow-y: auto;
    position: relative;
  }

  .sticky-header {
    position: sticky;
    top: 0;
    background-color: var(--color-background);
    z-index: 10;
    transition: transform 0.3s ease;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }

  .sticky-item, .ndc-item {
    padding: 6px 4px;
    font-size: var(--font-size-base);
  }

  .depth-filter select {
    height: 100%;
    padding: var(--spacing-small);
    font-size: var(--font-size-base);
    border: 1px solid var(--color-secondary);
    border-radius: 4px;
    background-color: var(--color-background);
    cursor: pointer;
    appearance: none;
    -webkit-appearance: none;
    -moz-appearance: none;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12'%3E%3Cpath fill='%23333' d='M10.293 3.293L6 7.586 1.707 3.293A1 1 0 00.293 4.707l5 5a1 1 0 001.414 0l5-5a1 1 0 10-1.414-1.414z'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 8px center;
    padding-right: 24px;
  }

  .depth-filter option {
    font-size: var(--font-size-base);
  }

  .depth-filter .unit {
    font-size: 0.7em;
    vertical-align: super;
  }

  .ndc-item:focus {
    border-radius: 4px;
    box-shadow: 0 0 0 3px rgba(var(--color-primary), 0.3);
    background-color: var(--color-secondary-light);
  }

  .ndc-item:hover {
    background-color: var(--color-hover);
  }

  .item-notation {
    font-weight: bold;
    margin-right: 6px;
  }

  .item-label {
    flex-grow: 1;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
</style>