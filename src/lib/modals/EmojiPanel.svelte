<script>
	import Modal from "../Modal.svelte";

	import {authHeader, chats, chat} from "../stores.js";
	import {apiUrl} from "../urls.js";
	import * as modals from "../modals.js";

    export let modalData;
    const postInput = modalData;
</script>

<Modal on:close={modals.closeLastModal}>
	<h2 slot="header">Emojis</h2>
	<div slot="default">
        {#each $chats as c}
            {#if c.owner}
                <h3>{c.nickname}</h3>
                <div class="emoji-panel">
                    {#each c.emojis as emoji}
                        <button class="emoji-button" on:click={()=>{
                             postInput.value = postInput.value + `<:${emoji._id}> `;
                             postInput.focus();
                             modals.closeLastModal();
                    	}}>
                            <img src={"https://uploads.meower.org/emojis/" + emoji._id} class="emoji-image" alt={emoji.name} />
                        </button>
                   {/each}
                   {#if c._id === $chat._id && c.owner === $authHeader.username}
                        <button class="emoji-button" on:click={()=>{
                    	}}>
                            <img src="$assets/add.svg" class="emoji-image" alt="Add Emoji" />
                        </button>
                   {/if}
               </div>
            {/if}
        {/each}
	</div>
</Modal>

<style>
    .emoji-panel {
        display: grid;
        grid-template-rows: 1fr 1fr 1fr 1fr 1fr;
        gap: 0.25em;
    }

    .emoji-button {
        background: #383645;
        width: 3em;
        height: 3em;
        border-radius: 10px;
        border: none;
        display: flex;
        justify-content: center;
        align-items: center;
    }

    .emoji-image {
        width: 2em;
        height: 2em;
    }
</style>
