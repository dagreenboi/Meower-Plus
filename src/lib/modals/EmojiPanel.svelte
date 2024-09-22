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
                <hr>
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
                            <img src="data:image/svg+xml,%3csvg%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%20width='16'%20height='16'%20viewBox='0,0,16,16'%3e%3cg%20transform='translate(-232,-172)'%3e%3cg%20data-paper-data='{&quot;isPaintingLayer&quot;:true}'%20fill='none'%20fill-rule='nonzero'%20stroke='%23ffffff'%20stroke-width='2'%20stroke-linecap='round'%20stroke-linejoin='round'%20stroke-miterlimit='10'%20stroke-dasharray=''%20stroke-dashoffset='0'%20style='mix-blend-mode:%20normal'%3e%3cpath%20d='M240,173v14'/%3e%3cpath%20d='M233,180h14'/%3e%3c/g%3e%3c/g%3e%3c/svg%3e%3c!--rotationCenter:8:8--%3e" class="emoji-image" alt="Add Emoji" />
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
        grid-template-columns: 1fr 1fr 1fr 1fr 1fr;
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
