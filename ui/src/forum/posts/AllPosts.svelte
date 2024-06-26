<script lang="ts">
import { onMount, getContext } from 'svelte';
import '@material/mwc-circular-progress';
import type { EntryHash, Record, AgentPubKey, ActionHash, AppAgentClient, NewEntryAction } from '@holochain/client';
import { clientContext } from '../../contexts';
import PostDetail from './PostDetail.svelte';
import type { PostsSignal } from './types';


let client: AppAgentClient = (getContext(clientContext) as any).getClient();

let hashes: Array<ActionHash> | undefined;
let loading = true;
let error: any = undefined;

$: hashes, loading, error;

onMount(async () => {

  await fetchPosts();
  client.on('signal', signal => {
    if (signal.zome_name !== 'posts') return;
    const payload = signal.payload as PostsSignal;
    if (payload.type !== 'EntryCreated') return;
    if (payload.app_entry.type !== 'Post') return;
    hashes = [...hashes, payload.action.hashed.hash];
  });
});

async function fetchPosts() {
  try {
    const links = await client.callZome({
      cap_secret: null,
      role_name: 'forum',
      zome_name: 'posts',
      fn_name: 'get_all_posts',
      payload: null,
    });
    hashes = links.map(l => l.target);
  } catch (e) {
    error = e;
  }
  loading = false;
}

</script>

{#if loading}
<div style="display: flex; flex: 1; align-items: center; justify-content: center">
  <mwc-circular-progress indeterminate></mwc-circular-progress>
</div>
{:else if error}
<span>Error fetching the posts: {error.data.data}.</span>
{:else if hashes.length === 0}
<mwc-button 
  raised
  label="Refresh"
  on:click={() => fetchPosts()}
></mwc-button>
<span>No posts found.</span>
{:else}
<div style="display: flex; flex-direction: column">
  <hr style="width: 100%; background-color: #000"/>
  <mwc-button 
  raised
  label="Refresh"
  on:click={() => fetchPosts()}
  ></mwc-button>
  {#each hashes as hash}
    <div style="margin-bottom: 8px;">
      <PostDetail postHash={hash}  on:post-deleted={() => fetchPosts()}></PostDetail>
    </div>
  {/each}
</div>
{/if}

