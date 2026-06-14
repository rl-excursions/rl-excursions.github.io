<script lang="ts">
  import { page } from "$app/stores";
  import { base } from "$app/paths";

  const logoMap: Record<string, { src: string; alt: string }> = {
    ucsd: { src: "/assets/figures/ucsd.png", alt: "UC San Diego" },
    mbzuai: { src: "/assets/figures/mbzuai.png", alt: "MBZUAI-IFM" },
    cmu: { src: "/assets/figures/cmu.png", alt: "Carnegie Mellon University" },
    harvard: { src: "/assets/figures/harvard.svg", alt: "Harvard University" },
  };

  const internshipLine = "Work done during internship at MBZUAI-IFM";
  const affiliationLogoMap: Record<string, { src: string; alt: string }> = {
    "UC San Diego": logoMap.ucsd,
    "MBZUAI-IFM": logoMap.mbzuai,
    "Carnegie Mellon University": logoMap.cmu,
    "Harvard University": logoMap.harvard,
  };

  type AuthorEntry = { name: string; url?: string; affils?: string[]; line?: number };

  const isAuthorEntry = (a: unknown): a is AuthorEntry =>
    !!a && typeof a === "object" && "name" in (a as AuthorEntry);

  $: authorData = $page.data.header?.authors;
  $: authorList = Array.isArray(authorData) && authorData.every(isAuthorEntry)
    ? (authorData as AuthorEntry[])
    : null;

  $: authorsLine1 = authorList?.filter((a) => (a.line ?? 1) === 1) ?? [];
  $: authorsLine2 = authorList?.filter((a) => a.line === 2) ?? [];

  const stripEqualStar = (name: string) => name.replace(/\*+$/, "").trim();
  const hasEqualStar = (name: string) => /\*$/.test(name);
  const stripDagger = (name: string) => name.replace(/†+$/, "").trim();
  const hasDagger = (name: string) => /†$/.test(name);
</script>

<header
  class="layout-xl justify-between items-start"
  data-sveltekit-noscroll
  data-sveltekit-preload-code="eager"
>
  <div class="header-inner mb-8">
    <h1 class="title-font font-bold text-black text-3xl mb-4 leading-tight">
      {$page.data.header?.title ??
        "RL Excursions during Pre-training: Re-examining Policy Optimization for LLM Training"}
    </h1>

    <div class="meta text-black">
      <div class="authors">
        {#if authorList}
          {#each [authorsLine1, authorsLine2] as authorLine}
            {#if authorLine.length > 0}
              <div class="author-line">
                {#each authorLine as author}
                  <span class="author">
                    {#if author.url}
                    <a href={author.url} target="_blank" rel="noopener noreferrer" class="author-name author-link">{stripDagger(stripEqualStar(author.name))}</a>
                  {:else}
                    <span class="author-name">{stripDagger(stripEqualStar(author.name))}</span>
                  {/if}
                    {#if author.affils}
                      {@const nonHarvardAffils = author.affils.filter((a) => a !== "harvard")}
                      {#if nonHarvardAffils.length > 0 || hasEqualStar(author.name) || hasDagger(author.name)}
                      <sup class="affil-sup">
                        {#each nonHarvardAffils as key, j (j)}
                          {#if logoMap[key]}
                            <img
                              src={base + logoMap[key].src}
                              alt={logoMap[key].alt}
                              title={logoMap[key].alt}
                              class={`affil-logo ${key === "mbzuai" ? "affil-logo--mbzuai" : ""} ${key === "cmu" ? "affil-logo--cmu" : ""} ${key === "harvard" ? "affil-logo--harvard" : ""}`}
                              loading="lazy"
                              decoding="async"
                            />
                          {:else if key === "intern"}
                            <span class="affil-dot" title={internshipLine} aria-label={internshipLine}></span>
                          {:else}
                            <span class="affil-fallback">{key}</span>
                          {/if}
                        {/each}
                        {#if hasEqualStar(author.name)}
                          <span class="affil-sup-text">*</span>
                        {/if}
                        {#if hasDagger(author.name)}
                          <span class="affil-sup-text">†</span>
                        {/if}
                      </sup>
                      {/if}
                    {/if}
                  </span>
                {/each}
              </div>
            {/if}
          {/each}
        {:else}
          {$page.data.header?.authors ?? "__AUTHORS__"}
        {/if}
      </div>

      <div class="affiliations">
        {#if Array.isArray($page.data.header?.affiliations) && $page.data.header.affiliations.length > 0}
          <div class="affiliation-line">
            {#each $page.data.header.affiliations as line, i (i)}
              <span class="affiliation-item">
                {#if line === internshipLine}
                  <span class="affil-dot" aria-hidden="true"></span>
                {:else if affiliationLogoMap[line]}
                  <img
                    src={base + affiliationLogoMap[line].src}
                    alt={affiliationLogoMap[line].alt}
                    title={affiliationLogoMap[line].alt}
                    class="affil-logo affil-logo--legend"
                    loading="lazy"
                    decoding="async"
                  />
                {/if}
                <span>{line}</span>
              </span>
              {#if i < $page.data.header.affiliations.length - 1}
                <span class="affiliation-sep"> </span>
              {/if}
            {/each}
          </div>
        {:else}
          {#if $page.data.header?.affiliations}
            {$page.data.header.affiliations}
          {/if}
        {/if}
        {#if $page.data.header?.correspondence}
          <div class="correspondence-line">{$page.data.header.correspondence}</div>
        {/if}
      </div>

      <div class="date">
        {$page.data.header?.date ?? "__DATE__"}
      </div>
    </div>
  </div>
</header>

<style lang="postcss">
  /* Keep the header aligned with the same main-text column width used by Markdown.svelte. */
  .header-inner {
    max-width: var(--md-main-col, 760px);
    margin-left: auto;
    margin-right: auto;
  }


  .title-font {
    font-family: "Iowan Old Style BT", "Iowan Old Style", "Palatino Linotype", Palatino, serif;
  }

  .meta {
    font-size: 16px;
    line-height: 1.55;
  }

  .authors {
    font-size: 16px;
    line-height: 2.0;
    text-align: center;
  }

  .author-line {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 4px 0;
    margin-bottom: 6px;
  }

  .author-line:last-child {
    margin-bottom: 0;
  }

  .author {
    display: inline-flex;
    align-items: baseline;
    gap: 1px;
    margin-right: 16px;
    margin-bottom: 4px;
    white-space: nowrap;
  }

  .author:last-child {
    margin-right: 0;
  }

  .author-name {
    line-height: 1.4;
  }

  .author-link {
    color: inherit;
    text-decoration: none;
  }

  .author-link:hover {
    text-decoration: underline;
  }

  .affil-sup {
    display: inline-flex;
    align-items: flex-start;
    gap: 0px;
    margin-left: 0px;
    transform: translateY(0px);
  }

  .affil-sup .affil-logo {
    margin-left: 0px;
  }

  .affil-sup-text {
    font-size: 15px;
    line-height: 1;
    margin-left: 0px;
    white-space: nowrap;
    position: relative;
    top: 8px;
  }
  .affil-sup .affil-dot {
    position: relative;
    margin-left: 0px;
    top: 8px;
  }

  .affil-logo {
    height: 16px;
    width: auto;
    vertical-align: middle;
    margin-left: 1px;
  }

  .affil-logo--mbzuai {
    height: 20px;
  }

  .affil-logo--cmu {
    height: 23px;
  }

  .affil-logo--harvard {
    height: 18px;
  }

  .affil-logo--legend {
    height: 22px;
    margin-right: 8px;
  }

  .affil-dot {
    width: 7px;
    height: 7px;
    border-radius: 999px;
    border: 1.5px solid #111827;
    background: transparent;
    display: inline-block;
    vertical-align: middle;
    margin-left: 2px;
    margin-right: 4px;
  }

  .affiliation-line .affil-dot {
    position: relative;
    top: -6px;
  }

  .affil-fallback {
    font-size: 12px;
    color: #6b7280;
    margin-left: 4px;
  }

  .affiliations {
    margin-top: 10px; /* visual separation from authors */
    color: #111827;
    opacity: 0.9;
    text-align: center;
  }

  .affiliation-line {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    justify-content: center;
    width: 100%;
  }

  .affiliation-item {
    display: inline-flex;
    align-items: center;
    gap: 4px;
  }

  .affiliation-sep {
    margin: 0 4px;
  }

  .affiliation-line {
    margin-top: 2px;
  }

  .correspondence-line {
    margin-top: 8px;
    font-size: 15px;
  }

  .date {
    margin-top: 10px;
    color: #111827;
    opacity: 0.9;
  }

  @media (max-width: 580px) {
    .meta {
      font-size: 15px;
    }
  }
</style>
