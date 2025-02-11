<script lang="ts">
    import {agent, columns, settings} from "$lib/stores";
    import TimerEvent from "$lib/components/utils/TimerEvent.svelte";
    import {createEventDispatcher} from "svelte";
    const dispatch = createEventDispatcher();

    export let column;
    export let index;
    export let _agent = $agent;
    export let unique = Symbol();

    let isRefreshing = false;

    export async function refresh() {
        isRefreshing = true;
        if (column.algorithm.type === 'default' || column.algorithm.type === 'custom' || column.algorithm.type === 'officialList') {
            const res = await _agent.getTimeline({limit: 20, cursor: '', algorithm: column.algorithm});
            const newFeeds = res.data.feed.filter(feed => {
                return !column.data.feed.some(item => feed.reason ? item.post.uri === feed.post.uri && item.reason?.indexedAt === feed.reason.indexedAt : item.post.uri === feed.post.uri);
            });

            if (newFeeds.length === 20) {
                await columns.update(_columns => {
                    _columns[index].data.feed = [...res.data.feed];
                    return _columns;
                })

                column.data.cursor = res.data.cursor;
            } else {
                const el = $settings.design?.layout === 'decks' ? column.scrollElement : document.querySelector(':root');
                const elInitialPosition = el.scrollTop;
                const topEl = el.querySelector('.timeline > .timeline__item:first-child');

                await columns.update(_columns => {
                    _columns[index].data.feed = [...newFeeds, ...column.data.feed];
                    return _columns;
                })

                if (elInitialPosition === 0 && column.settings?.refreshToTop !== true) {
                    if (column.style !== 'media') {
                        const offset = topEl.parentElement.getBoundingClientRect().top + 16;
                        el.scrollTo(0, topEl.getBoundingClientRect().top - offset);
                    }
                }
            }

            /* if (_agent.did() === $agent.did()) {
                notificationCount.set(await _agent.getNotificationCount());
            } */
            refreshNotificationCount();
        } else if (column.algorithm.type === 'bookmark') {
            column.data.feed = [];
            column.data.cursor = 0;
            unique = Symbol();
        } else if (column.algorithm.type === 'realtime') {
            return false;
        } else if (column.algorithm.type === 'notification') {
            $columns[index].unreadCount !== 0
                ? $columns[index].unreadCount = 0
                : await _agent.getNotificationCount();
            column.data.feed = [];
            column.data.cursor = undefined;
            unique = Symbol();
            await _agent.agent.api.app.bsky.notification.updateSeen({seenAt: new Date().toISOString()});
        } else {
            column.data.feed = [];
            column.data.cursor = undefined;
            unique = Symbol();
        }

        isRefreshing = false;
    }

    async function refreshNotificationCount() {
        for (const column1 of $columns) {
            const index1 = $columns.indexOf(column1);
            if (column1.algorithm.type === 'notification' && column1.did === _agent.did()) {
                $columns[index1].unreadCount = await _agent.getNotificationCount();
            }
        }
    }

    function handleTimer() {
        refresh();
    }

    function handleKeydown(event: { key: string; }) {
        const activeElement = document.activeElement?.tagName;

        if (event.key === 'r' && (activeElement === 'BODY' || activeElement === 'BUTTON') && !isRefreshing) {
            refresh();
        }
    }
</script>

<svelte:window on:keydown={handleKeydown} />

{#if (column.algorithm.type !== 'bookmark' && column.algorithm.type !== 'realtime')}
  <button
      class="refresh-button"
      class:refresh-button--decks={$settings.design.layout === 'decks'}
      class:is-refreshing={isRefreshing}
      aria-label="Refresh"
      on:click={() => {refresh()}}
      disabled={isRefreshing}
  >
    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="22.855" viewBox="0 0 16 22.855">
      <path id="refresh" d="M11,3.428V5.714a5.714,5.714,0,0,0-4.045,9.759L5.343,17.084A8,8,0,0,1,11,3.428Zm5.657,2.343A8,8,0,0,1,11,19.427V17.141a5.714,5.714,0,0,0,4.045-9.759ZM11,22.855,6.428,18.284,11,13.713ZM11,9.142V0L15.57,4.571Z" transform="translate(-2.999)" fill="var(--primary-color)"/>
    </svg>
  </button>
{/if}

{#if (column.settings?.autoRefresh && column.settings?.autoRefresh > 0)}
  <TimerEvent delay={Number(column.settings.autoRefresh) * 1000} on:timer={handleTimer}></TimerEvent>
{/if}