<script>
	import Container from "../lib/Container.svelte";
	import Loading from "../lib/Loading.svelte";
	import version from "../lib/version.js";
	import logo from "../assets/logo.svg";
	import meowy from "../assets/meowy.svg";

	import Contributor from "../lib/Contributor.svelte";

	const REPO_OWNER = "dagreenboi";
	const REPO_NAME = "Meower-Plus";

	async function fetchContributors() {
		const res = await fetch(
			`https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/contributors`
		);
		if (!res.ok) {
			throw new Error(
				`Received response code: ${res.status} ${res.statusText}`
			);
		}
		const json = await res.json();

		const shuffled = [];
		while (json.length > 0) {
			const index = Math.floor(Math.random() * json.length);
			shuffled.push(json[index]);
			json.splice(index, 1);
		}
		return shuffled;
	}
</script>

<h1 class="logo center">
	<img src={logo} alt="Meower" height="80" />
</h1>
<b class="center friendlier">The friendlier social media for everyone.</b>
<div class="center">Meower+ Client, version {version}</div>
<div class="center links">
	<a href="https://meower.org" target="_blank" rel="noreferrer">Learn more</a>
	|
	<a
		href="https://github.com/dagreenboi/Meower-Plus"
		target="_blank"
		rel="noreferrer">Source code</a
	>
</div>

<Container>
	<h2>Meower+ Contributors</h2>
	<p>These people are Cuntributors to the Meower+ Client.</p>
	<div class="contributors-list">
		<Contributor username="DaGreenBoi" pfp={26} isMeower={true}>
			Creator of Meower+
		</Contributor>
	</div>
</Container>
<Container>
	<h2>Github Contributors</h2>
	<p>(This list uses GitHub usernames/PFPs and the order is randomized.)</p>
	{#await fetchContributors()}
		<Loading />
	{:then contributors}
		<div class="contributors-list">
			{#each contributors as contributor}
				{#if contributor.type === "User"}
					<Contributor
						username={contributor.login}
						url={contributor.html_url}
						pfp={contributor.avatar_url}
					>
						{contributor.contributions} commit{contributor.contributions ===
						1
							? ""
							: "s"}
					</Contributor>
				{/if}
			{/each}
		</div>
	{:catch e}
		<p>An error occurred!</p>
		<pre><code>{e}</code></pre>
	{/await}
</Container>
<Container>
	<h2>Special Thanks</h2>
	<div class="contributors-list">
		<Contributor
			username="trappist-1e"
			url="https://scratch.mit.edu/users/trappist-1e/"
			pfp="https://uploads.scratch.mit.edu/get_image/user/61483990_100x100.png"
		>
			<img class="inline-meowy" src={meowy} height="24" alt="Meowy" /> Original
			creator of Meowy
		</Contributor>
	</div>
</Container>
<Container>
	<h2>Changelog</h2>
	<Container>
		<h2>0.2</h2>
		<ul>
            <li>Changed Replies to be Server-Side.</li>
            <li>Made Replies from Roarer display.</li>
        </ul>
	</Container>
	<Container>
		<h2>0.1</h2>
		<ul>
            <li>Added the ability to change your Profile Picture to any image.</li>
            <li>Added Profile Picture borders</li>
            <li>Changed Inbox a bit</li>
            <li>Updated the About page</li>
        </ul>
	</Container>
</Container>

<style>
	.center {
		text-align: center;
		display: block;
	}
	.logo {
		margin-bottom: 0;
	}
	.friendlier {
		font-size: 1.25em;
	}

	.links {
		margin-bottom: 1em;
	}

	#long {
		width: 100%;
	}

	.contributors-list {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(15em, 1fr));
		grid-gap: 0.5em;
	}

	.inline-meowy {
		vertical-align: middle;
	}

	h3 {
		margin-top: 0.5ex;
		margin-bottom: 0.5ex;
	}
</style>
