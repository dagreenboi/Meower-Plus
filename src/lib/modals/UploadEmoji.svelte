<script>
	import Modal from "../Modal.svelte";

	import {authHeader, chat} from "../stores.js";
	import {apiUrl} from "../urls.js";
	import * as modals from "../modals.js";

	import {goto, focus} from "@roxi/routify";

	let name, img, imgElement, loading, error;
  function uploadImg() {
    imgElement.click();
  }

  function imgChanged() {
    img = imgElement.files[0];
    let imgname = img.name.split(".");
    imgname.pop();
    imgname.join(".");
    name = imgname;
  }
</script>

<Modal on:close={modals.closeLastModal}>
	<h2 slot="header">Upload Emoji</h2>
	<div slot="default">
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
      <input type=file hidden bind:this={imgElement} on:change={imgChanged} accept="image/png,image/jpeg,image/webp,image/gif" />
			<div class="modal-buttons">
				<button
					type="button"
					disabled={loading}
					on:click={modals.closeLastModal}>Cancel</button
				>
        <button on:click={async () => {
				loading = true;

    const formData = new FormData();
    formData.append("file", img);
    const uploadsResp = await fetch("https://uploads.meower.org/emojis", {
        method: "POST",
        headers: { Authorization: $authHeader.token },
        body: formData,
    });
    const emojiId = (await uploadsResp.json()).id;
const apiResp = await fetch(`https://api.meower.org/chats/${$chat._id}/emojis/${emojiId}`, {
        method: "PUT",
        headers: {
            'Content-Type': 'application/json',
            token: $authHeader.token,
        },
        body: JSON.stringify({ name }),
    });
const cidx = $chats.indexOf($chat);
$chat.emojis = [...$chat.emojis,{_id: emojiId, name, animated: img.name.endsWith(".gif")}];
$chats[cidx] = $chat; 

                modals.closeLastModal();
            }} disabled={!(name && img)|| loading}
					>Create</button				
        >
			</div>
	</div>
</Modal>

<style>
	.long {
		width: 100%;
		margin: 0;
		margin-bottom: 0.2em;
  }
</style>
