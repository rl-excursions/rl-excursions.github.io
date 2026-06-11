<script lang="ts">
  import Seo from "$lib/components/Seo.svelte";
  import Markdown from "$lib/components/Markdown.svelte";
  import ScrollMeter from "$lib/components/ScrollMeter.svelte";
  import text from "../maintext/rl_excursions.md?raw";
</script>

<Seo
  title="RL Excursions during Pre-training: Re-examining Policy Optimization for LLM Training"
  description="We study when and how RL should be used in LLM training: direct RL vs SFT, sharpening vs expansion, general capability preservation, and parallel averaging."
/>

<div>
  <ScrollMeter containerSelector=".md-output" />

  <div class="layout-xl text-base space-y-12">
    <Markdown source={text} />
  </div>
</div>
