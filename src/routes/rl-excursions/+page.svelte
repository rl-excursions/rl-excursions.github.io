<!-- Legacy route: IsoCompute post. Not linked from main nav. Remove when no longer needed. -->
<script lang="ts">
  import Seo from "$lib/components/Seo.svelte";
  import Markdown from "$lib/components/Markdown.svelte";
  import ScrollMeter from "$lib/components/ScrollMeter.svelte";
  import type { PageData } from "./$types";

  export let data: PageData;
</script>

<Seo
  title="RL Excursions during Pre-training"
  description="RL Excursions during Pre-training: Re-examining Policy Optimization for LLM Training"
/>

<div>
  <ScrollMeter containerSelector=".md-output" />

  <div class="layout-xl text-base space-y-12">
    <Markdown source={data.text} />
  </div>
</div>

