<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import type {
    Agent,
    ComponentContext,
  } from '@ixon-cdk/types';

  export let context: ComponentContext;

  // !--- Local States ---!

  // Menu State of the open menu
  let openMenu: { type: 'name' | 'tags' | 'status', x: number, y: number } | null = null;
  let outputData: any[] | null = null;

  // Reactive input for the title 
  $: title = context?.inputs?.title || '';

  // Simplified sort states
  type SortKey = 'default' | 'name_asc' | 'name_desc' | 'tag_asc' | 'tag_desc' | 'status_last_online';
  let sortKey: SortKey = 'default';
  
  // !--- SVG filter icon ---!

  const filterIcon = `<svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" fill="currentColor" viewBox="0 0 16 16"><path d="M1.5 1.5A.5.5 0 0 1 2 1h12a.5.5 0 0 1 .5.5v2a.5.5 0 0 1-.128.334L10 8.692V13.5a.5.5 0 0 1-.74.439L7 12.439a.5.5 0 0 1-.26-.439V8.692L1.628 3.834A.5.5 0 0 1 1.5 3.5v-2z"/></svg>`;

  // !--- Lifecycle functions ---!
  
  onMount(() => {
    window.addEventListener('click', clickHandler);
    loadAgents();
  });
  
  onDestroy(() => {
    window.removeEventListener('click', clickHandler);
  });

  // !--- Menu Handlers ---!
  
  function toggleMenu(event: MouseEvent, menu: 'name' | 'tags' | 'status') {
    if (openMenu?.type === menu) {
      openMenu = null;
      return;
    }
    const btn = event.currentTarget as HTMLElement;
    const rect = btn.getBoundingClientRect();
    
    let x = rect.left;
    if (menu === 'tags') {
      x = rect.right;
    }

    openMenu = {
      type: menu,
      x: x,
      y: rect.bottom + 4
    };
  }

  function applySort(key: SortKey) {
    sortKey = key;
    openMenu = null;
    sortOutputData();
    outputData = [...outputData];
  }

  // Closes the menu if a click is outside
  function clickHandler(event: MouseEvent) {
    const target = event.target as HTMLElement;
    if (!target.closest('.filter-btn') && !target.closest('.sort-menu')) {
      openMenu = null;
    }
  }
  
  // !--- Headers for API calls ---!

  function getAuthorizationHeaders(
    context: ComponentContext,
  ): Record<string, string> {
    return {
      'Content-Type': 'application/json',
      Authorization: 'Bearer ' + context.appData.accessToken.secretId,
      'Api-Application': context.appData.apiAppId,
      'Api-Company': context.appData.company.publicId,
      'Api-Version': context.appData.apiVersion,
    };
  }

  // !--- Sorting and loading functions ---!

  function sortOutputData() {
    function compare(a, b) {
      switch (sortKey) {
        case 'name_asc':
          return a.name.localeCompare(b.name);
        
        case 'name_desc':
          return b.name.localeCompare(a.name);
        
        case 'tag_asc':
          const countA_asc = a.tags?.length ?? 0;
          const countB_asc = b.tags?.length ?? 0;
          return countA_asc - countB_asc;
        
        case 'tag_desc':
          const countA_desc = a.tags?.length ?? 0;
          const countB_desc = b.tags?.length ?? 0;
          return countB_desc - countA_desc;

        case 'status_last_online':
          if (a.isOnline && !b.isOnline) return 1;
          if (!a.isOnline && b.isOnline) return -1;
          if (!a.isOnline && !b.isOnline) {
            const dateA = a.lastSeenDate?.getTime() || 0;
            const dateB = b.lastSeenDate?.getTime() || 0;
            if (dateA === 0 && dateB !== 0) return -1;
            if (dateA !== 0 && dateB === 0) return 1;
            return dateA - dateB;
          }
          return a.name.localeCompare(b.name);

        default: // 'default'
          if (a.isOnline && !b.isOnline) return -1;
          if (!a.isOnline && b.isOnline) return 1;
          return a.name.localeCompare(b.name);
      }
    }
    if (outputData) {
      outputData.sort(compare);
    }
  }

  async function loadTags(context: ComponentContext, agent: Agent): Promise<any[]> {
    const tagList = [];
    let after = null;
    while (true) {
      const url = context.getApiUrl('AgentDataTagList', {
        agentId: agent.publicId,
        fields: 'source.publicId,tagId',
        'page-size': 250,
        ...(after === null ? {} : { 'page-after': after }),
      });
      const response = await fetch(url, {
        headers: getAuthorizationHeaders(context),
      });
      const responseJson = await response.json();
      tagList.push(...responseJson.data);
      if (responseJson.moreAfter === null) {
        break;
      }
      after = responseJson.moreAfter;
    }
    return tagList;
  }

  async function loadAgents() {
    const agentList = [];
    let after = null;
    while (true) {
      const url = context.getApiUrl('AgentList', {
        fields: 'name,dataSources(name),mdrServer,activeVpnSession,vpnChangedOn,mqttChangedOn',
        'page-size': 250,
        ...(after === null ? {} : { 'page-after': after }),
      });
      const response = await fetch(url, {
        headers: getAuthorizationHeaders(context),
      });
      const responseJson = await response.json();
      const locale = context.appData.locale || undefined;
      const processedAgents = responseJson.data.map(agent => {
        const isOnline = agent.mdrServer !== null || agent.activeVpnSession !== null;
        let status = '';
        let lastSeenDate: Date | null = null;
        if (isOnline) {
          status = 'Online';
        } else {
          const vpnDate = agent.vpnChangedOn ? new Date(agent.vpnChangedOn) : null;
          const mqttDate = agent.mqttChangedOn ? new Date(agent.mqttChangedOn) : null;
          if (vpnDate && mqttDate) {
            lastSeenDate = vpnDate < mqttDate ? vpnDate : mqttDate;
          } else {
            lastSeenDate = vpnDate || mqttDate;
          }
          if (lastSeenDate) {
            status = `Last seen online: ${lastSeenDate.toLocaleString(locale)}`;
          } else {
            status = 'Offline';
          }
        }
        return {
          publicId: agent.publicId,
          name: agent.name,
          dataSources: agent.dataSources,
          tags: undefined,
          status: status,
          isOnline: isOnline,
          lastSeenDate: lastSeenDate,
        };
      });
      agentList.push(...processedAgents);
      if (responseJson.moreAfter === null) {
        break;
      }
      after = responseJson.moreAfter;
    }
    outputData = agentList;
    sortOutputData();
    outputData = [...outputData];
    for (let agent of outputData) {
      agent.tags = await loadTags(context, agent);
      outputData = [...outputData];
    }
    loaded = true;
  }
</script>

<style lang="scss">
  @import './styles/main.scss';
</style>

<main>
  {#if title}
    <div class="component-title">{title}</div>
  {/if}
  {#if outputData === null}
    Loading agents...
  {:else}
    <div class="table-container">
      <table>
        <thead>
          <tr>
            <th>Agent ID</th>

            <th class="sortable">
              <!-- svelte-ignore a11y-click-events-have-key-events -->
              <div class="header-wrapper" on:click|stopPropagation>
                Agent name
                <button 
                  on:click={(e) => toggleMenu(e, 'name')} 
                  class:active={sortKey === 'name_asc' || sortKey === 'name_desc'}
                  class="filter-btn" 
                  title="Sort by name"
                >
                  {@html filterIcon}
                </button>
              </div>
            </th>

            <th class="sortable text-right">
              <!-- svelte-ignore a11y-click-events-have-key-events -->
              <div class="header-wrapper right" on:click|stopPropagation>
                Tags
                <button 
                  on:click={(e) => toggleMenu(e, 'tags')} 
                  class:active={sortKey === 'tag_asc' || sortKey === 'tag_desc'}
                  class="filter-btn" 
                  title="Sort by tag count"
                >
                  {@html filterIcon}
                </button>
              </div>
            </th>

            <th class="sortable">
              <!-- svelte-ignore a11y-click-events-have-key-events -->
              <div class="header-wrapper" on:click|stopPropagation>
                Status
                <button 
                  on:click={(e) => toggleMenu(e, 'status')} 
                  class:active={sortKey === 'status_last_online'}
                  class="filter-btn" 
                  title="Sort by status"
                >
                  {@html filterIcon}
                </button>
              </div>
            </th>
          </tr>
        </thead>
        <tbody>
          {#each outputData as agent}
            <tr class:offline={!agent.isOnline}>
              <td class="monospace">
                <a 
                  href={`https://portal.ixon.cloud/fleet-manager/device-configurator/${agent.publicId}/info`}
                  target="_blank"
                >
                  {agent.publicId}
                </a>
              </td>
              <td>{agent.name}</td>
              {#if agent.tags === undefined}
                <td class="text-center">(...)</td>
                <td class="text-center">(...)</td>
              {:else}
                <td class="text-right">{agent.tags.length}</td>
                <td>{agent.status}</td>
              {/if}
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
  {/if}

  {#if openMenu}
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <div 
      class="sort-menu" 
      class:align-right={openMenu.type === 'tags'}
      style="top: {openMenu.y}px; left: {openMenu.x}px;" 
      on:click|stopPropagation
    >
      {#if openMenu.type === 'name'}
        <button class:active={sortKey === 'name_asc'} class="menu-option" on:click={() => applySort('name_asc')}>
          A - Z
        </button>
        <button class:active={sortKey === 'name_desc'} class="menu-option" on:click={() => applySort('name_desc')}>
          Z - A
        </button>
        <button class:active={sortKey === 'default'} class="menu-option" on:click={() => applySort('default')}>
          Default
        </button>
      {:else if openMenu.type === 'tags'}
        <button class:active={sortKey === 'tag_desc'} class="menu-option" on:click={() => applySort('tag_desc')}>
          Highest to lowest
        </button>
        <button class:active={sortKey === 'tag_asc'} class="menu-option" on:click={() => applySort('tag_asc')}>
          Lowest to highest
        </button>
        <button class:active={sortKey === 'default'} class="menu-option" on:click={() => applySort('default')}>
          Default
        </button>
      {:else if openMenu.type === 'status'}
        <button class:active={sortKey === 'default'} class="menu-option" on:click={() => applySort('default')}>
          Online
        </button>
        <button class:active={sortKey === 'status_last_online'} class="menu-option" on:click={() => applySort('status_last_online')}>
          Last online
        </button>
      {/if}
    </div>
  {/if}
</main>