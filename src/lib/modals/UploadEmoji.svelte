<script>
	import Modal from "../../Modal.svelte";

	import {authHeader, chats} from "../../stores.js";
	import {apiUrl} from "../../urls.js";
	import * as modals from "../../modals.js";

	import {goto, focus} from "@roxi/routify";

	let name, img, imgElement, loading, error;
  function uploadImg() {
    imgElement.click();
  }

  function imgChanged() {
    img = imgElement.files[0];
    name = img.name;
  }
</script>

<Modal on:close={modals.closeLastModal}>
	<h2 slot="header">Upload Emoji</h2>
	<div slot="default">
		<form
			on:change={() => (error = "")}
			on:submit|preventDefault={async () => {
				loading = true;
                modals.closeLastModal();
            }}
		>
			<input
				id="nickname"
				type="text"
				class="modal-input white"
				placeholder="Name"
				maxlength="32"
				disabled={loading}
        bind:value={name}
				use:focus
			/><br />
      <br />
      <button
				class="long"
				title="Choose File"
        on:click={uploadImg}
			>
          Choose File
      </button>
      <input type=file hidden bind:this={imgElement} on:change={imgChanged} />
			<div class="modal-buttons">
				<button
					type="button"
					disabled={loading}
					on:click={modals.closeLastModal}>Cancel</button
				>
        <button type="submit" disabled={!(name && img)|| loading}
					>Create</button
				>
			</div>
		</form>
	</div>
</Modal>

<style>
	.long {
		width: 100%;
		margin: 0;
		margin-bottom: 0.2em;
  }
</style>
