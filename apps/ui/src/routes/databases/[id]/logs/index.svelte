<script lang="ts">
	import { onDestroy, onMount } from 'svelte';

	import { page } from '$app/stores';
	import LoadingLogs from './_Loading.svelte';
	import { get } from '$lib/api';
	import { t } from '$lib/translations';
	import { errorNotification } from '$lib/common';
	import Tooltip from '$lib/components/Tooltip.svelte';

	const { id } = $page.params;

	let loadLogsInterval: any = null;
	let logs: any = [];
	let lastLog: any = null;
	let followingInterval: any;
	let followingLogs: any;
	let logsEl: any;
	let position = 0;
	let loadingLogs = false;
	let database = {
		name: null
	};

	onMount(async () => {
		const response = await get(`/databases/${id}`);
		database = response.database;
		const { logs: firstLogs } = await get(`/databases/${id}/logs`);
		logs = firstLogs;
		loadAllLogs();
		loadLogsInterval = setInterval(() => {
			loadLogs();
		}, 1000);
	});
	onDestroy(() => {
		clearInterval(loadLogsInterval);
		clearInterval(followingInterval);
	});
	async function loadAllLogs() {
		try {
			loadingLogs = true;
			const data: any = await get(`/databases/${id}/logs`);
			if (data?.logs) {
				lastLog = data.logs[data.logs.length - 1];
				logs = data.logs;
			}
		} catch (error) {
			return errorNotification(error);
		} finally {
			loadingLogs = false;
		}
	}
	async function loadLogs() {
		if (loadingLogs) return;
		try {
			loadingLogs = true;
			const newLogs: any = await get(`/databases/${id}/logs?since=${lastLog?.split(' ')[0] || 0}`);

			if (newLogs?.logs && newLogs.logs[newLogs.logs.length - 1] !== logs[logs.length - 1]) {
				logs = logs.concat(newLogs.logs);
				lastLog = newLogs.logs[newLogs.logs.length - 1];
			}
		} catch (error) {
			return errorNotification(error);
		} finally {
			loadingLogs = false;
		}
	}
	function detect() {
		if (position < logsEl.scrollTop) {
			position = logsEl.scrollTop;
		} else {
			if (followingLogs) {
				clearInterval(followingInterval);
				followingLogs = false;
			}
			position = logsEl.scrollTop;
		}
	}

	function followBuild() {
		followingLogs = !followingLogs;
		if (followingLogs) {
			followingInterval = setInterval(() => {
				logsEl.scrollTop = logsEl.scrollHeight;
				window.scrollTo(0, document.body.scrollHeight);
			}, 1000);
		} else {
			clearInterval(followingInterval);
		}
	}
</script>

<div class="flex h-20 items-center space-x-2 p-5 px-6 font-bold">
	<div class="-mb-5 flex-col">
		<div class="md:max-w-64 truncate text-base tracking-tight md:text-2xl lg:block">
			Database Logs
		</div>
		<span class="text-xs">{database.name}</span>
	</div>
</div>
<div class="flex flex-row justify-center space-x-2 px-10 pt-6">
	{#if logs.length === 0}
		<div class="text-xl font-bold tracking-tighter">{$t('application.build.waiting_logs')}</div>
	{:else}
		<div class="relative w-full">
			<div class="text-right " />
			{#if loadLogsInterval}
				<LoadingLogs />
			{/if}
			<div class="flex justify-end sticky top-0 p-1 mx-1">
				<button
					id="follow"
					on:click={followBuild}
					class="bg-transparent btn btn-sm btn-link"
					class:text-green-500={followingLogs}
				>
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="w-6 h-6"
						viewBox="0 0 24 24"
						stroke-width="1.5"
						stroke="currentColor"
						fill="none"
						stroke-linecap="round"
						stroke-linejoin="round"
					>
						<path stroke="none" d="M0 0h24v24H0z" fill="none" />
						<circle cx="12" cy="12" r="9" />
						<line x1="8" y1="12" x2="12" y2="16" />
						<line x1="12" y1="8" x2="12" y2="16" />
						<line x1="16" y1="12" x2="12" y2="16" />
					</svg>
				</button>
				<Tooltip triggeredBy="#follow">Follow Logs</Tooltip>
			</div>
			<div
				class="font-mono w-full leading-6 text-left text-md tracking-tighter rounded bg-coolgray-200 py-5 px-6 whitespace-pre-wrap break-words overflow-auto max-h-[80vh] -mt-12 overflow-y-scroll scrollbar-w-1 scrollbar-thumb-coollabs scrollbar-track-coolgray-200"
				bind:this={logsEl}
				on:scroll={detect}
			>
				<div class="px-2 pr-14">
					{#each logs as log}
						{log + '\n'}
					{/each}
				</div>
			</div>
		</div>
	{/if}
</div>
