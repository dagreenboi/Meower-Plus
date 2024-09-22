<!--
	Component that powers lists of posts like home.
	Also customizable!

	Parameters:
	- fetchUrl: The API path to fetch (like "home" for /home). Adds ?autoget and ?page= by default.
		If falsy, a page won't be fetched by default (the list will start empty).
	- postOrigin: A post_origin to listen to for live post updates
		(for example, posts from home have the origin "home").
		If falsy, posts will not be listened to.
	- chatName: Only needed for group chats. Used for join/leave messages.
	- canPost: A boolean which indicates if this list can be posted in or not.
	- queryParams: Additional query parameters, as an object.

	Slots:
	- error and empty: These slots unction the same as PagedList (see its comment).
		There are no "loaded" or "item" slots.

	Events:
	- loaded: Fired when the list loads for the first time.
-->
<script>
	import BasicModal from "./modals/Basic.svelte";
	import AccountBannedModal from "./modals/safety/AccountBanned.svelte";
	import BlockUserModal from "./modals/safety/BlockUser.svelte";
	import SendFiles from "./modals/SendFiles.svelte";
	import EmojiPanel from "./modals/EmojiPanel.svelte";

	import {
		relationships,
		authHeader,
		user,
		lastTyped,
		chat,
	} from "./stores.js";
	import {userRestrictions, isRestricted} from "../lib/bitField.js";
	import {shiftHeld} from "./keyDetect.js";
	import {mobile} from "./responsiveness.js";
	import {playNotification} from "./sounds.js";
	import * as modals from "./modals.js";
	import PagedList from "./PagedList.svelte";
	import Container from "./Container.svelte";
	import Post from "./Post.svelte";
	import TypingIndicator from "./TypingIndicator.svelte";
	import ProfileView from "./ProfileView.svelte";
	import * as clm from "./clmanager.js";
	import {apiUrl, encodeApiURLParams} from "./urls.js";
    import { darkenColour, lightenColour } from "./color.js";

	import {createEventDispatcher} from "svelte";
	const dispatch = createEventDispatcher();

	// @ts-ignore
	import {autoresize} from "svelte-textarea-autoresize";

	import {params} from "@roxi/routify";
	import {onMount, onDestroy} from "svelte";

	export let fetchUrl = "home";
	export let postOrigin = "home";
	export let chatName = "Home";
	export let canPost = true;
	export let queryParams = {};
	export let addToChat = false;
	export let adminView = false;

	let id = 0;
	let postErrors = {};

	// User restriction status
	let userRestricted = false;
	$: {
		if (
			postOrigin === "home" &&
			isRestricted(userRestrictions.HOME_POSTS)
		) {
			userRestricted = true;
		} else if (
			!["home", "inbox"].includes(postOrigin) &&
			isRestricted(userRestrictions.CHAT_POSTS)
		) {
			userRestricted = true;
		} else {
			userRestricted = false;
		}
	}

	// DM recipient
	let dmWith = "";
	let recipientBlocked = false;
	$: {
		if ($chat.type === 1) {
			dmWith = $chat.members.find(username => username !== $user.name);
			recipientBlocked = $relationships[dmWith] === 2;
		}
	}

	// Elements
	let postInput, submitBtn;

	let addToChatLoading = {};

	// PagedList stuff
	let list;
	export let items = [];
	let firstLoad = true;

	/**
	 * Loads a page.
	 * @param {number} page
	 * @returns {Promise<{
	 * 	numPages: number,
	 * 	result: Array<import("./types.js").ListPost | import("./types.js").User>
	 * }>}
	 */
	async function loadPage(page = 1) {
		if (!fetchUrl) {
			// don't load anything
			if (firstLoad) dispatch("loaded");
			firstLoad = false;
			return {
				numPages: 1,
				result: [],
			};
		}

		let result;

		const params = new URLSearchParams({
			autoget: "1",
			page: page.toString(),
			...queryParams,
		}).toString();

		let path = `${fetchUrl}?${params}`;
		if (encodeApiURLParams) path = encodeURIComponent(path);
		const resp = await fetch(`${apiUrl}${path}`, {
			headers: $authHeader,
		});

		if (!resp.ok) {
			throw new Error("Response code is not OK; code is " + resp.status);
		}
		/**
		 * @type {import("./types.js").ServerPostList}
		 */
		const json = await resp.json();

		let posts = [];
		if (json.autoget) {
			posts = json.autoget;
		} else {
			posts = [json];
		}

		/**
		 * @type {Array<import("./types.js").ListPost | import("./types.js").User>}
		 */
		result = posts.map(post => {
			if ("lower_username" in post) {
				/** @type import("./types.js").User */
				// @ts-ignore
				const user = {
					id: id++,
					...post,
				};
				return user;
			}
			return {
				id: id++,
				post_id: post._id,
				post_origin: post.post_origin,
				user: post.u,
				content: post.p,
				attachments: post.attachments,
				date: post.t.e,
				edited_at: post.edited_at,
				isDeleted: post.isDeleted,
				mod_deleted: post.mod_deleted,
				deleted_at: post.deleted_at,
                reply_to: post.reply_to
			};
		});
		if ($user.hide_blocked_users) {
			// @ts-ignore
			result = result.filter(
				post =>
					$relationships[post._id] !== 2 &&
					$relationships[post.u] !== 2
			);
		}
		const numPages = json["pages"];

		if (firstLoad) dispatch("loaded");
		firstLoad = false;
		return {
			numPages,
			result,
		};
	}

	function sendPost(content) {
		/**
		 * @type {import('src/lib/types.js').ListPost}
		 */
		const pendingPost = {
			id: id++,
			nonce: Math.random().toString(),
			post_origin: postOrigin,
			user: $user.name,
			content,
			attachments: [],
			date: Math.floor(new Date().getTime() / 1000),
			isDeleted: false,
			pending: true,
            reply_to: replies
		};
		list.addItem(pendingPost);

		const postProm = new Promise(async (resolve, reject) => {
			try {
				const resp = await fetch(
					`${apiUrl}${
						postOrigin === "home" ? "home" : `/posts/${$chat._id}`
					}`,
					{
						method: "POST",
						headers: {
							"Content-Type": "application/json",
							...$authHeader,
						},
						body: JSON.stringify({
							content,
							nonce: pendingPost.nonce,
                            reply_to: replyids
						}),
					}
				);
				const json = await resp.json();
                replyids = [];
                replies = [];

				if (resp.ok) {
					resolve();
				} else {
					reject(json.type);
				}
			} catch (e) {
				reject(e);
			}
		});
		postProm.catch(e => {
			postErrors[pendingPost.id] = e;
		});
	}

    let isDark = false;

	onMount(() => {
        isDark = document.getElementById("main").classList.contains("mode-dark");
		if (postOrigin) {
			const evId = clm.link.on("direct", cmd => {
				if (!cmd.val) return;

				const isGC = postOrigin !== "home";
				if (cmd.val.mode === "delete") {
					if (adminView) return;
					items = items.filter(post => post.post_id !== cmd.val.id);
				}
				if (cmd.val.mode === "update_post") {
					let itemIndex = items.findIndex(
						post => post.post_id === cmd.val.payload._id
					);
					if (itemIndex !== -1) {
						let post = cmd.val.payload;
						items[itemIndex] = {
							id: items[itemIndex].id,
							post_id: post._id,
							post_origin: post.post_origin,
							user: post.u,
							content: post.p,
							attachments: post.attachments,
							date: post.t.e,
							edited_at: post.edited_at,
							isDeleted: post.isDeleted,
							mod_deleted: post.mod_deleted,
							deleted_at: post.deleted_at,
                            reply_to: post.reply_to
						};
					}
				}
				if (!isGC || cmd.val.state === 2) {
					if (cmd.val.post_origin !== postOrigin) return;
					const post = {
						id: id++,
						post_id: cmd.val._id,
						post_origin: cmd.val.post_origin,
						user: cmd.val.u,
						content: cmd.val.p,
						attachments: cmd.val.attachments,
						date: cmd.val.t.e,
						edited_at: cmd.val.edited_at,
						isDeleted: cmd.val.isDeleted,
						mod_deleted: cmd.val.mod_deleted,
						deleted_at: cmd.val.deleted_at,
                        reply_to: cmd.val.reply_to
					};
					list.addItem(post);
					if (cmd.val.nonce) {
						items = items.filter(post => post.nonce !== cmd.val.nonce);
					}
					if ($user.sfx && cmd.val.u !== $user.name)
						playNotification();
				}
			});
			destroy = () => clm.link.off(evId);
		}
	});

	let destroy = () => {};
	onDestroy(() => destroy());

    function handleReplyLink(_id) {
        const postElement = document.getElementById(_id);
        console.log(postElement)
        if (postElement) {
            const postRect = postElement.getBoundingClientRect();
            const containerRect = document.getElementById("posts-list").getBoundingClientRect();
            const postPosition = postRect.top - containerRect.top + document.scrollingElement.scrollTop;
            console.log(postPosition)
            document.scrollingElement.scrollTo({
                top: postPosition,
                behavior: "smooth"
			});
        }
    }

    let replies = [];
    let replyids = [];

    async function handleReplyPost(_id) {
        if (!replyids.includes(_id) && replyids.length < 10) {
            let replyingPost = await fetch(`https://api.meower.org/posts?id=${_id}`, {
                headers: { token: $authHeader.token }
            });
            replyingPost = replyingPost.status === "404" ? { p: "[original message was deleted]" } : await replyingPost.json();
            replies = [...replies,replyingPost];
            replyids = [...replyids,_id];
            postInput.focus();
        }
    }
</script>

<div>
	<!-- I think we discussed that guest posting will not be in
		the official client, due to moderation reasons -->
	{#if canPost && $user.name}
		<form
			class="createpost"
			autocomplete="off"
			on:submit|preventDefault={/** @type {SubmitEvent} */ e => {
				// @ts-ignore
				const input = e.target.elements.input;
				// @ts-ignore
				const content = e.target.elements.input.value.trim();
				if (!content) {
					return;
				}

				if (content.match(/^s\/.+\//gs)) {
					// substitute command
					let post = items.find(_post => _post.user === $user.name);
					if (!post) return;

					let toReplace = content.split("/")[1];
					let replaceWith = content.replace(`s/${toReplace}/`, "");

					let newContent = post.content;
					newContent = newContent
						.replace(toReplace, replaceWith)
						.trim();

					if (newContent) {
						fetch(`${apiUrl}posts?id=${post.post_id}`, {
							method: "PATCH",
							headers: {
								"Content-Type": "application/json",
								...$authHeader,
							},
							body: JSON.stringify({
								content: newContent,
							}),
						}).catch(e => {
							console.error(e);
							postErrors[post.id] = e;
						});
					} else {
						fetch(`${apiUrl}posts?id=${post.post_id}`, {
							method: "DELETE",
							headers: $authHeader,
						}).catch(e => (postErrors[post.id] = e));
					}
				} else {
					// normal post
					sendPost(content);
				}

				input.value = "";
				input.rows = "1";
				input.style.height = "45px";
			}}
		>
            {#if (!userRestricted) && (!recipientBlocked)}
				<button
					class="emoji-panel-button"
					title="Open Emoji Panel"
					on:click|preventDefault={() => {
                        modals.showModal(EmojiPanel, postInput);
					}}
                >
                    <svg width="20" height="20" viewBox="0 0 16 16" fill="none" xmlns="http://www.w3.org/2000/svg">
                        <path d="M4 9.77778C4 9.77778 5.33333 10.2222 8 10.2222C10.6667 10.2222 12 9.77778 12 9.77778C12 9.77778 11.1111 11.5556 8 11.5556C4.88889 11.5556 4 9.77778 4 9.77778Z" fill="currentColor"></path>
                        <path fill-rule="evenodd" clip-rule="evenodd" d="M16 8C16 12.4184 12.4183 16 8 16C3.58171 16 0 12.4184 0 8C0 3.5816 3.58171 0 8 0C12.4183 0 16 3.5816 16 8ZM8 9.33377C6.38976 9.33377 5.32134 9.14627 4 8.88932C3.69824 8.83116 3.11111 8.88932 3.11111 9.77821C3.11111 11.556 5.15332 13.7782 8 13.7782C10.8462 13.7782 12.8889 11.556 12.8889 9.77821C12.8889 8.88932 12.3018 8.83073 12 8.88932C10.6787 9.14627 9.61024 9.33377 8 9.33377ZM5.33333 7.55556C5.94699 7.55556 6.44444 6.85894 6.44444 6C6.44444 5.14106 5.94699 4.44444 5.33333 4.44444C4.71967 4.44444 4.22222 5.14106 4.22222 6C4.22222 6.85894 4.71967 7.55556 5.33333 7.55556ZM11.7778 6C11.7778 6.85894 11.2803 7.55556 10.6667 7.55556C10.053 7.55556 9.55556 6.85894 9.55556 6C9.55556 5.14106 10.053 4.44444 10.6667 4.44444C11.2803 4.44444 11.7778 5.14106 11.7778 6Z" fill="currentColor"></path>
                    </svg>
                </button>
            {/if}
			<textarea
				type="text"
				class="white"
				placeholder={userRestricted
					? "Your account is currently restricted."
					: recipientBlocked
					? `You have blocked ${dmWith}.`
					: "Write something..."}
				name="input"
				autocomplete="off"
				maxlength="4000"
				rows="1"
				disabled={userRestricted || recipientBlocked}
				use:autoresize
				on:input={() => {
					if ($lastTyped + 2000 < +new Date()) {
						lastTyped.set(+new Date());
						fetch(
							`${apiUrl}${
								postOrigin === "home"
									? "home"
									: `/chats/${$chat._id}`
							}/typing`,
							{
								method: "POST",
								headers: $authHeader,
							}
						);
					}
				}}
				on:keydown={event => {
					if (
						event.key == "Enter" &&
						!shiftHeld &&
						!($mobile && shiftHeld)
					) {
						event.preventDefault();
						if (!submitBtn.disabled) submitBtn.click();
					}
				}}
				bind:this={postInput}
			/>
			{#if userRestricted}
				<button
					on:click|preventDefault={() => {
						modals.showModal(AccountBannedModal, {
							ban: $user.ban,
							feature: `creating ${
								postOrigin === "home" ? "home" : "group chat"
							} posts`,
						});
					}}>View details</button
				>
			{:else if recipientBlocked}
				<button
					on:click|preventDefault={() =>
						modals.showModal(BlockUserModal, {username: dmWith})}
					>Unblock</button
				>
			{:else}
				<button
					class="send-files"
					title="Send files"
					on:click|preventDefault={() => {
						modals.showModal(SendFiles, postOrigin);
					}}>+</button
				>
				<button
					bind:this={submitBtn}
					name="submit"
					disabled={!postInput}>Post</button
				>
			{/if}
		</form>
	{/if}
    {#each replies as reply}
        <div class="replyto-container">
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
            <button class="circle close" on:click={()=>{
                const replyidx = replyids.indexOf(reply._id);
                replyids = [...replyids.slice(0,replyidx), ...replyids.slice(replyidx + 1)];
                replies = [...replies.slice(0,replyidx), ...replies.slice(replyidx + 1)];
			}}></button>
        </div>
    {/each}
	{#if postOrigin}
		<TypingIndicator forPage={postOrigin} />
	{/if}
	<PagedList id="posts-list" maxItems={100} bind:items {loadPage} bind:this={list}>
		<svelte:fragment slot="loaded" let:items={_items}>
			{#each _items as post (post.id)}
				<div class="item">
					{#if "lower_username" in post}
						<ProfileView
							profile={post}
							small={true}
							canClick={true}
							canDoActions={!addToChat}
						/>
						{#if addToChat && !$chat.members.includes(post._id)}
							<div class="settings-controls">
								<button
									class="circle add"
									title="Add to chat"
									disabled={addToChatLoading[post._id]}
									on:click={async () => {
										addToChatLoading[post._id] = true;
										try {
											const resp = await fetch(
												`${apiUrl}${
													$params.admin
														? "admin/"
														: ""
												}chats/${$chat._id}/members/${
													post._id
												}`,
												{
													method: "PUT",
													headers: $authHeader,
												}
											);
											if (!resp.ok) {
												switch (resp.status) {
													case 403:
														throw new Error(
															`Someone's privacy settings are preventing you from adding ${post._id} to ${$chat.nickname}.`
														);
													case 404:
														throw new Error(
															`${post._id} not found.`
														);
													case 409:
														throw new Error(
															`${post._id} is already a member of ${$chat.nickname}.`
														);
													case 429:
														throw new Error(
															"Too many requests! Try again later."
														);
													default:
														throw new Error(
															"Response code is not OK; code is " +
																resp.status
														);
												}
											}
											if ($params.admin) {
												$chat = await resp.json();
											}
											delete addToChatLoading[post._id];
										} catch (e) {
											modals.showModal(BasicModal, {
												title: `Failed to add ${post._id} to ${$chat.nickname}`,
												desc: e,
											});
											delete addToChatLoading[post._id];
										}
									}}
								/>
							</div>
						{/if}
					{:else}
						<Post
							{post}
							{adminView}
							error={postErrors[post.id]}
                            input={postInput}
							retryPost={() => {
								sendPost(post.content);
								items = items.filter(v => v.id !== post.id);
							}}
							removePost={() =>
								(items = items.filter(v => v.id !== post.id))}
                            gotoRepliedToPost={handleReplyLink}
                            replyPost={handleReplyPost}
						/>
					{/if}
				</div>
			{/each}
		</svelte:fragment>
		<slot name="error" slot="error" let:error {error}>
			<Container>
				Error loading posts. Please try again.
				<pre><code>{error}</code></pre>
			</Container>
		</slot>
		<slot name="empty" slot="empty">
			{#if postOrigin !== "livechat"}
				<Container>
					{#if $user.name && canPost}
						No posts here. Check back later or be the first to post!
					{:else}
						No posts here. Check back later!
					{/if}
				</Container>
			{/if}
		</slot>
	</PagedList>
</div>

<style>
	.createpost {
		top: 0;
		display: flex;
		margin-bottom: 0.5em;
		gap: 0.25em;
	}
	.createpost textarea {
		flex-grow: 1;
		resize: none;
		max-height: 300px;
	}

	.send-files {
		padding: 0;
		padding-left: 0.8rem;
		padding-right: 0.8rem;
		font-size: 2rem;
	}

	.settings-controls {
		position: absolute;
		top: 0.5em;
		right: 0.5em;
	}

	.item {
		position: relative;
	}

	button.circle {
		border: none;
		margin: 0;
		margin-left: 0.125em;
	}

    .replyto-container {
        width: 100%;
        margin-bottom: .5em;
        display: flex;
        gap: .8rem;
        height: 48px;
        align-items: center;
        flex-direction: row;
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
        width: 80%;
    }

    .emoji-panel-button {
        width: 42.7037px;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }
</style>
