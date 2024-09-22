<script>
	import Modal from "../Modal.svelte";

	import {authHeader, chats} from "../stores.js";
	import {apiUrl} from "../urls.js";
	import * as modals from "../modals.js";

    export let modalData;
    const postInput = modalData;
    const Chats = $chats;
</script>

<Modal on:close={modals.closeLastModal}>
	<h2 slot="header">Emojis</h2>
	<div slot="default">
        {#each Chats as chat}
            <h3>{chat.nickname}</h3>
            <div class="emoji-panel">
                {#each chat.emojis as emojj}
                    <button class="emoji-button" on:click={()=>{
                         postInput.value = postInput.value + `<:${emoji._id}> `;
                         postInput.focus();
                         modals.closeLastModal();
                	}}></button>
               {/each}
           </div>
        {/each}
	</div>
</Modal>

<style>
    .emoji-panel {
        display: grid;
        grid-template-rows: 1fr 1fr 1fr 1fr 1fr;
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
