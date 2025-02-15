<!-- A post. Profile pictures not appearing while not logged in is intentional. -->
<script>
	import Container from "../lib/Container.svelte";
	import PFP from "../lib/PFP.svelte";
	import FormattedDate from "./FormattedDate.svelte";
	import Badge from "./Badge.svelte";
	import LiText from "./LiText.svelte";
	import Attachment from "./Attachment.svelte";
	import ExternalImage from "./ExternalImage.svelte";

	import ConfirmHyperlinkModal from "./modals/ConfirmHyperlink.svelte";
	import DeletePostModal from "./modals/DeletePost.svelte";
	import ReportPostModal from "./modals/safety/ReportPost.svelte";
	import ModeratePostModal from "./modals/moderation/ModeratePost.svelte";

	import {authHeader, user, chat, ulist} from "../lib/stores.js";
	import {adminPermissions, hasPermission} from "../lib/bitField.js";
	import {apiUrl} from "./urls.js";
	import {shiftHeld} from "../lib/keyDetect.js";
	import * as modals from "./modals.js";
    import { darkenColour, lightenColour } from "./color.js";

	import {IMAGE_HOST_WHITELIST} from "./hostWhitelist.js";

	import {default as loadProfile} from "../lib/loadProfile.js";

	// @ts-ignore
	import {autoresize} from "svelte-textarea-autoresize";
	import MarkdownIt from "markdown-it";
	import twemoji from "@twemoji/api";
	import {goto} from "@roxi/routify";
	import {onMount, tick} from "svelte";

	/**
	 * @type {import('src/lib/types.js').ListPost}
	 */
	export let post = {
		content: ""
	};
	export let buttons = true;
	export let adminView = false;
	export let error = "";
    export let input = null;
	export let retryPost;
	export let removePost;
    export let gotoRepliedToPost;
    export let replyPost;

	let bridged = false;
	let webhook = false;

	let images = [];

	let editing = false;
	let editContentInput, editSaveButton;

	let deleteButton;

	let adminDeleteButton, adminRestoreButton;

	// TODO: make bridged tag a setting

	/**
	 * Initialize this post's special behavior (user profile, images)).
	 */
	export async function initPostUser() {
		if (!post.user) return;

		if (post.content.includes(":")) {
			bridged =
				post.user === "Discord" ||
				post.user === "revolt" ||
				post.user === "Revower";
			webhook = post.user == "Webhooks";
		}

		if ((bridged || webhook) && post.content) {
			if (webhook) {
				post.content = post.content.slice(
					post.content.indexOf(": ") + 2
				);
			}
			post.user = post.content.split(": ")[0];
			post.content = post.content.slice(post.content.indexOf(": ") + 2);
		}

		if (!webhook) loadProfile(post.user);
	}

	$: {
		// Match image syntax
		// ([title: https://url])
		const iterator = post.content.matchAll(
			/\[([^\]]+?): (https:\/\/[^\]]+?)\]/gs
		);
		images = [];
		while (true) {
			const result = iterator.next();
			if (result.done) break;

			try {
				new URL(result.value[2]);
			} catch (e) {
				continue;
			}

			if (
				IMAGE_HOST_WHITELIST.some(o =>
					result.value[2].toLowerCase().startsWith(o.toLowerCase())
				)
			) {
				images.push({
					title: result.value[1],
					url: result.value[2],
				});

				if (images.length >= 3) break;
			}
		}
	}

	function confirmLink(link) {
		link = atob(encodeURIComponent(link)); // we now base64 encode the link to prevent XSS
		if (
			!link.startsWith("https:") &&
			!link.startsWith("http:") &&
			!link.startsWith("/")
		) {
			link = "https://" + link;
		}
		let url = new URL(link, location.href);
		if (url.host === location.host) {
			$goto(url.pathname);
		} else {
			modals.showModal(ConfirmHyperlinkModal, {link: url.href});
		}
		return false;
	}
	// @ts-ignore
	window.confirmLink = confirmLink;

	function createEmote(original, id, isGif, size) {
		return `​![${original}](https://cdn.discordapp.com/emojis/${id}.${isGif ? "gif" : "webp"}?size=32&quality=lossless):(emoji)`
	}

	/**
	 * @param {string} content
	 * @returns {Promise<{
	 * 	html?: string,
	 * 	error?: Error,
	 * 	youtubeEmbeds?: Array<{title: string, author: string, thumbnail: string, id: number, url: string}>
	 * }>}
	 */
	async function addFancyElements(content) {
		// markdown (which has HTML escaping built-in)
		let renderedContent;
		const youtubeEmbeds = [];

		try {
			const md = new MarkdownIt("default", {
				breaks: true,
				linkify: true,
				typographer: true,
			});
			md.linkify.add("@", {
				validate: function (text, pos) {
					let tail = text.slice(pos);
					return tail.match(/[a-zA-Z0-9-_]{0,20}/gs)[0].length;
				},
				normalize: function (match) {
					match.url = "/users/" + match.url.replace(/^@/, "");
				},
			});

			var emoteContent = content;
            const discordEmotes = Array.from(content.matchAll(/<a?:\w+:\d+>/g))

            discordEmotes.forEach(element => {
				const isGif = element[0].startsWith("<a:")
                const tok = element[0].replace("a", '').slice(0, -1).slice(1).split(":")

                emoteContent = emoteContent.split(element).join(createEmote(element, tok[2], isGif))
            });

            var emojiContent = emoteContent;
            const meowerEmojis = Array.from(content.matchAll(/<:(\w+)>/g))
            meowerEmojis.forEach(element => {
                emojiContent = emojiContent.replace(element[0],`![Emoji](https://uploads.meower.org/emojis/${element[1]}):(emoji)`);
			});


			const tokens = md.parse(
				emojiContent
					.replaceAll(/\[([^\]]+?): (https:\/\/[^\]]+?)\]/gs, "")
					.replaceAll(/\*\*\*\*/gs, "\\*\\*\\*\\*"),
				{}
			);
			for (const token of tokens) {
				if (token.children) {
					for (const childToken of token.children) {
						if (childToken.type === "image") {
							const srcPos = childToken.attrs.findIndex(
								attr => attr[0] === "src"
							);
							if (
								!IMAGE_HOST_WHITELIST.some(o =>
									childToken.attrs[srcPos][1]
										.toLowerCase()
										.startsWith(o.toLowerCase())
								)
							) {
								childToken.attrs[srcPos][1] = "about:blank";
							}
						}
						if (childToken.type === "link_open") {
							const href = childToken.attrs.find(
								attr => attr[0] === "href"
							)[1];
							const b64href = decodeURIComponent(btoa(href));
							childToken.attrs.push([
								"onclick",
								`return confirmLink('${b64href}')`,
								`return confirmLink('${b64href}')`,
							]);
						}
					}
				}
			}

			renderedContent = md.renderer.render(tokens, md.options);

			// add quote containers to blockquotes (although, quotes are currently broken)
			renderedContent = renderedContent.replaceAll(
				/<blockquote>(.+?)<\/blockquote>/gms,
				'<blockquote><p>$1</p></blockquote>'
			);
            // make emoji have class
            renderedContent = renderedContent.replaceAll(
				/<img src="(.+?)" alt="(.+?)">:\(emoji\)/gms,
				'<img src="$1" alt="$2" class="emoji">'
			);

			if ($user.embeds_enabled) {
				const youtubeMatches = [
					...new Set(
						content.match(
							/(\<|)(http|https):\/\/(www.youtube.com\/watch\?v=|youtube.com\/watch\?v=|youtu.be\/)(\S{11})(\>|)?/gm
						)
					),
				];
				for (const watchURL of youtubeMatches) {
					if (watchURL.startsWith("<") && watchURL.endsWith(">"))
						continue;
					const response = await (
						await fetch(
							`https://youtube.com/oembed?url=${watchURL}`,
							{cache: "force-cache"},
						)
					).json();
					youtubeEmbeds.push({
						title: response.title,
						author: response.author_name,
						thumbnail: response.thumbnail_url,
						id: youtubeEmbeds.length,
						url: watchURL
					});
				}
			}
		} catch (e) {
			// this is to stop any possible XSS attacks by bypassing the markdown lib
			// which is responsible for escaping HTML
			return {error: e};
		}

		// twemoji
		renderedContent = twemoji.parse(renderedContent, {
			folder: "svg",
			ext: ".svg",
			size: 20,
		});

		return {
			html: renderedContent,
			youtubeEmbeds,
		};
	}

	async function adminDelete() {
		adminDeleteButton.disabled = true;
		try {
			const resp = await fetch(`${apiUrl}admin/posts/${post.post_id}`, {
				method: "DELETE",
				headers: $authHeader,
			});
			if (!resp.ok) {
				throw new Error(
					"Response code is not OK; code is " + resp.status
				);
			}
			post.isDeleted = true;
			post.mod_deleted = true;
			post.deleted_at = Math.floor(Date.now() / 1000);
		} catch (e) {
			error = e;
		}
		adminDeleteButton.disabled = false;
	}

	async function adminRestore() {
		adminRestoreButton.disabled = true;
		try {
			const resp = await fetch(
				`${apiUrl}admin/posts/${post.post_id}/restore`,
				{
					method: "POST",
					headers: $authHeader,
				}
			);
			if (!resp.ok) {
				throw new Error(
					"Response code is not OK; code is " + resp.status
				);
			}
			post.isDeleted = false;
			post.mod_deleted = null;
			post.deleted_at = null;
		} catch (e) {
			error = e;
		}
		adminRestoreButton.disabled = false;
	}

    let isDark = false;

	onMount(async () => {
        isDark = document.getElementById("main").classList.contains("mode-dark");
		initPostUser();
	});

	$: noPFP =
		post.user === "Notification" ||
		post.user.startsWith("Notification to ") ||
		post.user === "Announcement" ||
		post.user === "Server" ||
		webhook;

    const roarer = /@([\w-]+)\s+"([^"]*)"\s+\(([^)]+)\)/g;
    const bettermeower = /@([\w-]+)\[([a-zA-Z0-9]+)\]/g;

    let matches1 = [...post.content.matchAll(roarer)];
    let matches2 = [...post.content.matchAll(bettermeower)];

    let allMatches = matches1.concat(matches2);
    let postsreplied = [];

    if (allMatches.length > 0) {
        const replyIds = allMatches.map(match => match[3] || match[2]);
        const roarRegex = /^@[\w-]+ (.+?) \(([^)]+)\)/;
        replyIds.forEach(async replyid => {
            let replydata;
            const replyresp = await fetch(`https://api.meower.org/posts?id=${replyid}`, {
                headers: { token: $authHeader.token }
            });
            if (replyresp.status === 404) {
                replydata = { p: "[original message was deleted]" };
            } else {
                replydata = await replyresp.json();
            }

            let content;
            let user;

            if (replydata.p) {
                content = replydata.p;

                let match = replydata.p.replace(roarRegex, "").trim();

                if (match) {
                    content = match;
                }
            } else if (replydata.attachments) {
                content = "[Attachment]";
            } else {
                content = '';
            }

            user = replydata.u || '';
            postsreplied = [...postsreplied, {_id: replydata._id || "", p: content, author: {_id: user}}];
	    });
       
        allMatches.forEach(match => {
            post.content = post.content.replace(match[0], '').trim();
        });
    }
</script>

<Container id={post.post_id}>
	<div class="post-header">
		{#if buttons}
			<div class="settings-controls">
				{#if post.pending}
					{#if error}
						<button
							class="circle restore"
							title="Retry"
							on:click={retryPost}
						/>
						<button
							class="circle trash"
							title="Delete"
							on:click={removePost}
						/>
					{/if}
				{:else if adminView && hasPermission(adminPermissions.DELETE_POSTS)}
					{#if post.isDeleted}
						<button
							class="circle restore"
							title="Restore post"
							bind:this={adminRestoreButton}
							on:click={adminRestore}
						/>
					{:else}
						<button
							class="circle trash"
							title="Delete post"
							bind:this={adminDeleteButton}
							on:click={adminDelete}
						/>
					{/if}
				{:else if !adminView}
					{#if !editing && hasPermission(adminPermissions.VIEW_POSTS)}
						<button
							class="circle admin"
							on:click={() =>
								modals.showModal(ModeratePostModal, {
									postid: post.post_id,
								})}
						/>
					{/if}
					{#if input && !input.disabled && !noPFP && !editing}
						{#if post.user === $user.name}
							{#if !bridged}
								<button
									class="circle pen"
									on:click={async () => {
										error = "";
										editing = true;
										await tick();
										editContentInput.value = post.content;
										editContentInput.focus();
										autoresize(editContentInput);
									}}
								/>
							{:else}
								<button
									class="circle pen"
									disabled
									title="You cannot edit bridged messages."
								/>
							{/if}
						{/if}
						<button
							class="circle reply"
							on:click={() => {replyPost(post.post_id)}}
						/>
						{#if post.user === $user.name || (post.post_origin === $chat._id && $chat.owner === $user.name)}
							{#if !bridged}
								<button
									class="circle trash"
									bind:this={deleteButton}
									on:click={async () => {
										if (shiftHeld) {
											deleteButton.disabled = true;
											try {
												const resp = await fetch(
													`${apiUrl}posts?id=${post.post_id}`, {
														method: "DELETE",
														headers: $authHeader,
													}
												);
												if (!resp.ok) {
													if (resp.status === 429) {
														throw new Error(
															"Too many requests! Try again later."
														);
													}
													throw new Error(
														"Response code is not OK; code is " +
															resp.status
													);
												}
											} catch (e) {
												error = e;
											}
											deleteButton.disabled = false;
										} else {
											modals.showModal(DeletePostModal, {
												post,
											});
										}
									}}
								/>
							{:else}
								<button
									class="circle trash"
									disabled
									title="You cannot delete bridged messages."
								/>
							{/if}
						{:else}
							<button
								class="circle report"
								on:click={() =>
									modals.showModal(ReportPostModal, {post})}
							/>
						{/if}
					{/if}
				{/if}
			</div>
		{/if}
		<button
			class="pfp"
			on:click={async () => {
				if (noPFP) return;
				$goto(`/users/${post.user}`);
			}}
		>
			{#await noPFP ? Promise.resolve(true) : loadProfile(post.user)}
				<PFP
					icon={-2}
					alt="{post.user}'s profile picture"
					online={$ulist.includes(post.user)}
				/>
			{:then profile}
				<PFP
					icon={post.user === "Server"
					    ? -4
						: profile.pfp_data}
					alt="{post.user}'s profile picture"
					online={$ulist.includes(post.user)}
                    data={profile}
				/>
			{:catch}
				<PFP
					icon={-2}
					alt="{post.user}'s profile picture"
					online={$ulist.includes(post.user)}
				/>
			{/await}
		</button>
		<div class="creatordate">
			<div class="creator">
				<h2>
					<LiText text={post.user} />
				</h2>

				{#if bridged}
					<Badge
						text="BRIDGED"
						title="This post is bridged from an external service by a bot"
					/>
				{/if}

				{#if webhook}
					<Badge
						text="WEBHOOK"
						title="This post was posted by the @Webhooks bot. The username may not mean the user actually posted it!"
					/>
				{/if}

				<!-- disabled until proper bot badges are added
				{#if post.isvbot && !webhook}
					<Badge
						text="BOT"
						title="This bot has been verified"
						checkmark={true}
					/>
				{/if}

				{#if post.isuvbot && !webhook}
					<Badge text="BOT" title="This bot has not been verified" />
				{/if}
				-->
			</div>

			<FormattedDate date={post.date} />
			{#if post.edited_at}
				<i
					>(<FormattedDate
						date={post.edited_at}
						customText="edited"
					/>)</i
				>
			{/if}
			{#if post.isDeleted && post.mod_deleted}
				<i
					>(<FormattedDate
						date={post.deleted_at}
						customText="deleted by moderator"
					/>)</i
				>
			{:else if post.isDeleted}
				<i
					>(<FormattedDate
						date={post.deleted_at}
						customText="self-deleted"
					/>)</i
				>
			{/if}
		</div>
	</div>
	{#if editing}
		<textarea
			type="text"
			class="white"
			name="input"
			autocomplete="off"
			maxlength="4000"
			rows="1"
			bind:this={editContentInput}
			use:autoresize
			on:keydown={event => {
				if (event.key == "Enter" && !shiftHeld) {
					event.preventDefault();
					if (!editSaveButton.disabled) editSaveButton.click();
				} else if (event.key == "Escape") {
					editing = false;
				}
			}}
		/>
		<div style="display: flex; justify-content: space-between;">
			<button on:click={() => (editing = false)}>Cancel</button>
			<button
				bind:this={editSaveButton}
				on:click={async () => {
					if (editContentInput.value.trim() === "") {
						modals.showModal(DeletePostModal, {post});
						return;
					}

					editing = false;
					try {
						const resp = await fetch(
							`${apiUrl}posts?id=${post.post_id}`,
							{
								method: "PATCH",
								headers: {
									"Content-Type": "application/json",
									...$authHeader,
								},
								body: JSON.stringify({
									content: editContentInput.value,
								}),
							}
						);
						if (!resp.ok) {
							if (resp.status === 429) {
								throw new Error(
									"Too many requests! Try again later."
								);
							}
							throw new Error(
								"Response code is not OK; code is " +
									resp.status
							);
						}
					} catch (e) {
						error = e;
					}
				}}>Save</button
			>
		</div>
	{:else}
        <div>
            {#each postsreplied as reply}
                <div
                    class="reply-container"
                    on:click={()=>{gotoRepliedToPost(reply._id)}}
                >
                    <p style="font-weight:bold;margin: 10px 0 10px 0;">{reply.author._id}</p>
                    <p style="margin: 10px 0 10px 0;">{reply.p}</p>
                </div>
            {/each}
            {#each post.reply_to as reply}
                <div
                    class="custom reply-container"
                    style:--reply-accent={(reply || {}).author ? (isDark ? darkenColour(reply.author.avatar_color, 3) : lightenColour(reply.author.avatar_color, 2)) : ""}
                    style:--reply-border={(reply || {}).author ? (isDark ? lightenColour(reply.author.avatar_color, 3) : "#" + reply.author.avatar_color) : ""}
                    style:--reply-color={(reply || {}).author ? (isDark ? lightenColour(reply.author.avatar_color, 1.5) : darkenColour(reply.author.avatar_color, 2)) : ""}
                    on:click={()=>{gotoRepliedToPost(reply._id)}}
                >
                    <p style="font-weight:bold;margin: 10px 0 10px 0;">{(reply || {}).author ? reply.author._id : ""}</p>
                    <p style="margin: 10px 0 10px 0;">{reply ? (reply.p ? reply.p : reply.attachments ? "Attachment" : "") : "[original message was deleted]"}</p>
                </div>
            {/each}
        </div>
		<p
			class="post-content"
			style="border-left-color: #4b5563; {post.pending
				? 'opacity: 0.5;'
				: ''}"
		>
			{#await addFancyElements(post.content) then content}
				{#if content.error}
					Error rendering post: {content.error}
				{:else}
					{#if content.html}
						{@html content.html}
					{/if}
					{#if content.youtubeEmbeds}
						{#each content.youtubeEmbeds as embed (embed.id)}
							<div class="youtube-container">
								<h4>{embed.title}</h4>
								<span style="color: #606060; font-size: 1.1rem"
									>{embed.author}</span
								><br /><br />
								<a href={embed.url} target="_blank" rel="noreferrer">
									<img
										src={embed.thumbnail}
										height="180"
										loading="lazy"
										alt="Thumbnail for {embed.title}"
									/>
								</a>
							</div>
						{/each}
					{/if}
				{/if}
			{/await}
		</p>
	{/if}
	{#if error}
		<p style="color: crimson;">{error}</p>
	{/if}
	{#if !editing}
		<div class="post-images">
			{#each post.attachments as attachment}
				<Attachment
					id={attachment.id}
					filename={attachment.filename}
					mime={attachment.mime}
				/>
			{/each}
			{#each images as { title, url }}
				<ExternalImage {title} {url} />
			{/each}
		</div>
	{/if}
</Container>

<style>
	.pfp {
		margin-right: 0.2em;
		padding: 0;
		border: none;
		background: none !important;
		color: inherit;
	}
	.post-header {
		display: flex;
		align-items: center;
		flex-wrap: wrap;
		max-width: 100%;
	}

	.creatordate {
		margin-left: 0.5em;
	}
	.creator {
		display: flex;
		flex-wrap: wrap;
		align-items: center;
		gap: 0.5em;
	}
	.creator h2 {
		font-size: 200%;
		margin: 0;
		overflow-wrap: anywhere;
	}
	:global(main.input-hover) .pfp:hover:not(:active) :global(.pfp),
	:global(main.input-touch) .pfp:active :global(.pfp),
	.pfp:focus-visible :global(.pfp) {
		transform: scale(1.1);
	}
	.settings-controls {
		position: absolute;
		top: 0.25em;
		right: 0.25em;
	}
	button.circle {
		border: none;
		margin: 0;
		margin-left: 0.125em;
	}

	.post-images {
		display: flex;
		flex-wrap: wrap;
		gap: 0.25em;
	}

	textarea {
		margin-top: 0.5em;
		margin-bottom: 0.5em;
		width: 100%;
	}

	:global(.youtube-container) {
		border: solid 2px var(--orange);
		padding: 10px;
		border-radius: 10px;
		width: 50%;
	}


	:global(blockquote) {
		margin: 0.25em;
		border-left: 4px solid var(--orange-dark);
	}

	:global(blockquote p) {
		margin-left: 1em;
	}
	:global(blockquote blockquote) {
		margin-left: 1em;
	}

    .reply-container.custom {
        background-color: var(--reply-accent);
        border: 3px solid var(--reply-border);
        color: var(--reply-color);
    }

    .reply-container {
        font-size: 16px;
        background-color: var(--accent-down);
        border: 3px solid var(--accent-down);
        padding-left: 10px;
        padding-right: 10px;
        border-radius: 6px;
        max-height: 100px;
        overflow-x: auto;
        overflow-y: visible;
        margin: 10px 0;
        display: flex;
        text-wrap: nowrap;
        gap: 10px;
        align-items: center;
        height: 42px;
        white-space: nowrap;
        overflow: hidden;
    }
</style>
