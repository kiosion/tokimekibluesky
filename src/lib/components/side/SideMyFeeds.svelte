<script lang="ts">
    import {agent} from "$lib/stores";
    import {liveQuery} from "dexie";
    import {db} from "$lib/db";
    import {createEventDispatcher, onMount} from "svelte";
    import {Bookmark, List, Newspaper} from "lucide-svelte";
    import {_} from "svelte-i18n";
    import LoadingSpinner from "$lib/components/ui/LoadingSpinner.svelte";
    const dispatch = createEventDispatcher();

    export let _agent = $agent;

    type tab = 'all' | 'feeds' | 'lists' | 'bookmarks';
    let currentTab: tab = 'all';
    let customFeeds = [];
    let officialLists = [];
    let bookmarks = liveQuery(() => db.bookmarks.toArray());
    let loaded = false;

    async function updateLists() {
        const res = await _agent.agent.api.app.bsky.graph.getLists({actor: _agent.did() as string, limit: 100, cursor: ''});
        officialLists = res.data.lists;
    }

    async function updateFeeds() {
        customFeeds = await $agent.getSavedFeeds();
    }

    function getFeedUrl(uri, genre = 'feed') {
        if (!uri) {
            return false;
        }

        const did = uri.split('/')[2];
        const name = uri.split('/')[4];
        return '/profile/' + did + '/' + genre + '/' + name;
    }

    function selectCategory(genre) {
        currentTab = genre;
    }

    function handleSelect() {
        dispatch('close');
    }

    onMount(async () => {
        await Promise.all([updateFeeds(), updateLists()]);
        loaded = true;
    })
</script>

<div class="side-feeds">
  <div class="side-feeds-nav">
    <ul class="profile-tab profile-tab--small">
      <li class="profile-tab__item" on:click={() => {selectCategory('all')}} class:profile-tab__item--active={currentTab === 'all'}><button>{$_('all')}</button></li>

      <li class="profile-tab__item" on:click={() => {selectCategory('feeds')}} class:profile-tab__item--active={currentTab === 'feeds'}><button>{$_('feeds')}</button></li>

      <li class="profile-tab__item" on:click={() => {selectCategory('lists')}} class:profile-tab__item--active={currentTab === 'lists'}><button>{$_('lists')}</button></li>

      <li class="profile-tab__item" on:click={() => {selectCategory('bookmarks')}} class:profile-tab__item--active={currentTab === 'bookmarks'}><button>{$_('bookmarks_short')}</button></li>
    </ul>
  </div>

  {#if loaded}
    <div class="side-feeds-content">
      {#if (currentTab === 'feeds' || currentTab === 'all')}
        <div class="side-feeds-list" data-category="feeds">
          {#if customFeeds.length}
            {#each customFeeds as feed}
              <li class="side-feeds-list__item">
                <a class="side-feeds-list__link" href="{getFeedUrl(feed.uri, 'feed')}" on:click={handleSelect}>
                  <Newspaper color="var(--text-color-3)" size="20"></Newspaper>
                  {feed.name}</a>
              </li>
            {/each}
          {/if}
        </div>
      {/if}

      {#if (currentTab === 'lists' || currentTab === 'all')}
        <div class="side-feeds-list" data-category="lists">
          {#if officialLists.length}
            {#each officialLists as list}
              <li class="side-feeds-list__item">
                <a class="side-feeds-list__link" href="{getFeedUrl(list.uri, 'lists')}" on:click={handleSelect}>
                  <List color="var(--text-color-3)" size="20"></List>
                  {list.name}</a>
              </li>
            {/each}
          {/if}
        </div>
      {/if}

      {#if (currentTab === 'bookmarks' || currentTab === 'all')}
        <div class="side-feeds-list" data-category="bookmarks">
          {#if ($bookmarks)}
            {#each $bookmarks as bookmark}
              {#if (bookmark.owner === $agent.did())}
                <li class="side-feeds-list__item">
                  <a class="side-feeds-list__link" href="/bookmark/{bookmark.id}" on:click={handleSelect}>
                    <Bookmark color="var(--text-color-3)" size="20"></Bookmark>
                    {bookmark.name}</a>
                </li>
              {/if}
            {/each}
          {/if}
        </div>
      {/if}
    </div>
  {:else}
    <LoadingSpinner size="32"></LoadingSpinner>
  {/if}
</div>

<style lang="postcss">
  .side-feeds {

  }

  .side-feeds-nav {
      color: var(--text-color-3);
  }

  .side-feeds-content {
      padding: 16px;
  }

  .side-feeds-list {
      list-style: none;
      display: grid;
      gap: 12px;
      margin-bottom: 12px;

      &__item {
          position: relative;
      }

      &__link {
          height: 40px;
          display: flex;
          align-items: center;
          padding: 0 12px;
          border: 1px solid var(--border-color-2);
          border-radius: var(--border-radius-2);
          color: var(--text-color-1);
          box-shadow: 0 0 6px var(--box-shadow-color-2);
          font-weight: bold;
          gap: 8px;
          letter-spacing: .025em;
      }
  }
</style>