<script>
	import {
		languageTag,
		onSetLanguageTag,
		setLanguageTag,
		sourceLanguageTag
	} from '$paraglide/runtime';
	import { onDestroy, onMount } from 'svelte';
	import { browser } from '$app/environment';
	import { getCookie, deleteCookie } from '$lib/utils/cookies';

	onMount(() => {
		// const valueFromSession = sessionStorage.getItem('lang') || sourceLanguageTag;
		const valueFromCookies = getCookie('lang') || sourceLanguageTag;
		// @ts-ignore
		setLanguageTag(valueFromCookies);
	});

	onDestroy(() => {
		if (browser) {
			deleteCookie('lang');
			// sessionStorage.removeItem('lang');
		}
	});

	// initialize the language tag
	$: _languageTag = languageTag;

	onSetLanguageTag((newLanguageTag) => {
		// @ts-ignore
		_languageTag = newLanguageTag;
	});
</script>

{#key _languageTag}
	<slot />
{/key}
