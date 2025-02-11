<script lang="ts">
    import {agent, pulseRepost} from '$lib/stores';
    import toast from 'svelte-french-toast';
    import { _ } from 'svelte-i18n';
    import { createEventDispatcher } from 'svelte';

    const dispatch = createEventDispatcher();

    export let _agent = $agent;
    export let cid;
    export let uri;
    export let count;
    export let repostViewer;
    export let showCounts = true;
    let timeoutId;

    let isProcessed: boolean = false;
    $: handleLikeChange($pulseRepost);

    function handleLikeChange(data) {
        if (!data) {
            return false;
        }

        if (uri === data.uri) {
            count = data.count;
            repostViewer = data.viewer;

            dispatch('repost', {
                count: data.count,
                viewer: data.viewer,
            });
        }
    }

    export async function repost(cid: string, uri: string, repostViewer) {
        isProcessed = true;

        if (!repostViewer) {
            count = count + 1;
        } else {
            count = count - 1;
        }

        pulseRepost.set({
            uri: uri,
            count: count,
            viewer: repostViewer
        })

        try {
            const repost = await _agent.setRepost(cid, uri, repostViewer || '');

            try {
                isProcessed = false;
                repostViewer = repost?.uri || undefined;

                pulseRepost.set({
                    uri: uri,
                    count: count,
                    viewer: repostViewer
                })

                toast.success($_('success_to_repost_or_delete_repost'));

                if (timeoutId) {
                    clearTimeout(timeoutId);
                }
                timeoutId = setTimeout(() => {
                    pulseRepost.set(undefined);
                }, 1000)
            } catch(e) {
                toast.error($_('failed_to_repost_after_reload'));
                console.log(e)
                isProcessed = false;
            }
        } catch (e) {
            toast.error($_('failed_to_repost'));
            console.error(e);
            isProcessed = false;

            if (!repostViewer) {
                count = count - 1;
            } else {
                count = count + 1;
            }

            pulseRepost.set({
                uri: uri,
                count: count,
                viewer: repostViewer
            })
        }
    }
</script>

<button class="timeline-reaction__item timeline-reaction__item--repost" disabled="{isProcessed}" on:click="{() => repost(cid, uri, repostViewer)}">
  <span class="timeline-reaction__icon" aria-label="リポスト">
    {#if (repostViewer)}
      <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="var(--primary-color)" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-repeat-2"><path d="m2 9 3-3 3 3"/><path d="M13 18H7a2 2 0 0 1-2-2V6"/><path d="m22 15-3 3-3-3"/><path d="M11 6h6a2 2 0 0 1 2 2v10"/></svg>
    {:else}
      <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="var(--timeline-reaction-repost-icon-color)" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-repeat-2"><path d="m2 9 3-3 3 3"/><path d="M13 18H7a2 2 0 0 1-2-2V6"/><path d="m22 15-3 3-3-3"/><path d="M11 6h6a2 2 0 0 1 2 2v10"/></svg>
    {/if}
  </span>

  {#if showCounts}
    <span class="timeline-reaction__count">{ count || 0 }</span>
  {/if}
</button>

<style lang="postcss">
    .timeline-reaction__item {
        &:hover {
            @media (min-width: 768px) {
                color: var(--timeline-reaction-repost-icon-hover-color);

                .timeline-reaction__icon::after {
                    background-color: var(--timeline-reaction-repost-hover-bg-color);
                }

                path {
                    stroke: var(--timeline-reaction-repost-icon-hover-color);
                }
            }
        }
    }
</style>