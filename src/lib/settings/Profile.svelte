<script>
	import ProfileView from "../ProfileView.svelte";
	import Loading from "../Loading.svelte";
	import Container from "../Container.svelte";
	import PFP from "../PFP.svelte";

	import {user, authHeader} from "../stores.js";
	import {userRestrictions, isRestricted} from "../bitField.js";
	import {profileCache} from "../loadProfile.js";
	import {apiUrl, encodeApiURLParams, uploadsUrl} from "../urls.js";
	import * as clm from "../clmanager.js";

	import {params} from "@roxi/routify";

    let pfpElement;
    let profileColorElement;

    function changeProfilePicture() {
        pfpElement.click();
    }

    function changeProfileColor() {
        profileColorElement.click();
    }

    function onPfpChange() {
        const file = pfpElement.files[0];
        const formData = new FormData();
        formData.append("file",file);
        fetch(`${uploadsUrl}icons`, {
            method: "POST",
            headers: {
                "Authorization": $authHeader.token
            },
            body: formData
		})
        .then(uploadResponse => uploadResponse.json())
        .then(uploadData => {
            const avatarId = uploadData.id;
            $user.avatar = avatarId;
            clm.updateProfile({avatar: $user.avatar});
		})
        .catch(error => console.error("Error uploading file:",error));
    }

    function onProfileColorChange() {
        $user.avatar_color = profileColorElement.value.substring(1);
        clm.updateProfile({avatar_color: $user.avatar_color});
    }

	async function loadProfile() {
		let path = `users/${$user.name}`;
		if (encodeApiURLParams) path = encodeURIComponent(path);
		const resp = await fetch(`${apiUrl}${path}`);
		if (!resp.ok) {
			throw new Error("Response code is not OK; code is " + resp.status);
		}
		const json = await resp.json();
		return json;
	}

	let pfpOverflow = false;
	$: {
		const pfp = $user.pfp_data;
	}
</script>

<div>
	{#await loadProfile()}
		<div class="fullcenter">
			<Loading />
		</div>
	{:then data}
		<ProfileView profile={data} />

		<Container>
			<h3>Quote</h3>
			<input
				type="text"
				class="modal-input white"
				style="font-style: italic; margin-bottom: 0.6em;"
				placeholder="Write something..."
				maxlength="360"
				disabled={isRestricted(userRestrictions.EDITING_QUOTE)}
				bind:value={$user.quote}
				on:change={() => clm.updateProfile({quote: $user.quote})}
			/>
            <h3>Profile Picture</h3>
            <button
				class="long"
				title="Choose File"
				on:click={changeProfilePicture}
                style="margin-bottom: 0.6em;"
			>
                Choose File
            </button>
            <input type=file hidden bind:this={pfpElement} on:change={onPfpChange} accept="image/png,image/jpeg,image/webp,image/gif" />
            <h3>Profile Color</h3>
            <button
				class="long"
				title="Choose Color"
				on:click={changeProfileColor}
                style="background: #{$user.avatar_color}; color: #{$user.avatar_color}; user-select: none;"
			>
                Choose Color
            </button>
            <input type=color hidden bind:this={profileColorElement} on:change={onProfileColorChange} />
		</Container>
	{:catch}
		<ProfileView username={$params.username} />
	{/await}
</div>

<style>
	.fullcenter {
		text-align: center;
		display: flex;
		justify-content: center;
		align-items: center;

		width: 100vw;
		height: 100vh;

		position: fixed;
		top: 0;
		left: 0;
	}

	.long {
		width: 100%;
		margin: 0;
		margin-bottom: 0.2em;
	}

	:global(main.input-hover) .pfp:hover,
	:global(main.input-touch) .pfp:active {
		background-color: var(--orange-light);
	}
	/* cst todo: fix shadow appearing when activating then unhovering because i gtg*/
	:global(:root main.input-hover) .pfp:active {
		background-color: var(--orange-dark);
	}
</style>
