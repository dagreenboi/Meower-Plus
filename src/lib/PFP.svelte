<!-- To nobody's surprise, a profile picture! -->
<script>
	export let icon = -1;
	export let alt = "Profile picture";
	export let online = false;
	export let size = 1;
	export let raw = false;
    export let data = {};
    export let chat = false;
    data = data || {};

	import errorIcon from "../assets/avatars/icon_err.svg";
    import GCIcon from "../assets/GC.svg";

	/**
	 * @type {*}
	 */
	const icons = import.meta.glob("../assets/avatars/*.svg", {
		import: "default",
		eager: true,
	});

	// only respond to `icon` changing
	let id;
	function setId(val) {
		id = val;
	}
	$: setId(icon);
</script>

{#if !chat}
<span on:click class:pfp-container={!raw} style:--size={size} style:--avatar-color={data.avatar_color ? "#" + data.avatar_color : "#ffffff"}>
	<span class:pfp={!raw} class:raw-pfp={raw}>
		<img 
            class="pfp-img"
			{alt}
			title={alt}
			src={
                data.avatar
                ?
                "https://uploads.meower.org/icons/" + data.avatar
                :
                icons[
				`../assets/avatars/icon_${
					id === -1
						? 21
						: id === -2
						? "err"
						: id === -3
						? "guest"
                        : id === -4
                        ? id
						: id - 1
				}.svg`
			] || errorIcon}
			on:error={() => (id = -2)}
			class:loading={icon === -1}
			draggable={false}
			width="auto"
			height="100%"
		/>
	</span>
</span>
{:else}
<span on:click class:pfp-container={!raw} style:--size={size} style:--avatar-color={data.icon_color ? "#" + data.icon_color : "#ffffff"}>
	<span class:pfp={!raw} class:raw-pfp={raw}>
		<img 
            class="pfp-img"
			{alt}
			title={alt}
			src={
                data.icon
                ?
                "https://uploads.meower.org/icons/" + data.icon
                :
                GCIcon
			|| errorIcon}
            draggable={false}
			width="auto"
			height="100%"
		/>
	</span>
</span>
{/if}

<style>
	.pfp-container {
		display: inline-block;
		position: relative;
	}
	.raw-pfp {
		width: calc(var(--size) * 3.75em);
		height: calc(var(--size) * 3.75em);
	}
	.pfp {
		width: calc(var(--size) * 3.75em);
		height: calc(var(--size) * 3.75em);
		box-sizing: border-box;

		background-color: var(--avatar-color);
		border: solid 3px var(--avatar-color);
		border-radius: 100%;

		display: flex;
		align-items: center;
		justify-content: center;
		user-select: none;

		/* Always make fallback text visible */
		color: black;
	}

    .pfp-img {
		width: 100%;
        height: 100%;
        border-radius: 100%;
    }

	.loading {
		animation: spin 0.5s linear infinite;
		filter: saturate(0) brightness(1.5);
	}
	@keyframes spin {
		from {
			transform: rotate(0turn);
		}
		to {
			transform: rotate(1turn);
		}
	}

	.online {
		display: inline-block;

		width: calc(var(--size) * 1em);
		height: calc(var(--size) * 1em);
		border-radius: 100%;

		background-color: limegreen;

		position: absolute;
		bottom: calc(var(--size) * -0.2em);
		right: calc(var(--size) * -0.2em);
		z-index: 1;
	}
</style>
