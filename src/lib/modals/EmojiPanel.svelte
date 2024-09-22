<script>
	import Modal from "../../Modal.svelte";

	import {authHeader, chats} from "../../stores.js";
	import {apiUrl} from "../../urls.js";
	import * as modals from "../../modals.js";
</script>

<Modal on:close={modals.closeLastModal}>
	<h2 slot="header">Emojis</h2>
	<div slot="default">
    <h3>E</h3>
    <button class="emoji-button">
      <img src="https://uploads.meower.org/emojis/TRxWqE2g9Wb7pljLBGEH63pj" class="emoji-image" />
    </button>
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
