<script lang="ts">
	import '../app.css';

	let { children } = $props();
</script>

<div class="container max-w-5xl mx-auto h-screen">
	<div class="flex-auto h-full content-center">
		<h1 class="scroll-m-20 text-4xl tracking-tight lg:text-5xl inline-flex mb-1.5">
			Konf<div class="font-bold pl-0.5">fusion</div>
		</h1>
		{@render children()}
	</div>
</div>