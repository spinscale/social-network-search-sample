<script>
  export let hit;
  export let border = 'border-gray-500'
</script>

{#if hit}
  <div class="{border} bg-white rounded-lg p-3 border w-1/2 mb-3 flex">
    <div class="h-24 w-24 flex-none bg-cover rounded-t lg:rounded-t-none lg:rounded-l text-center" style="background-image: url('/images/{hit._source.image}.png')">
    </div>

    <div class="ml-8">
      <h2 class="text-lg">{hit._source.name.full}</h2>
      <div class="text-teal-500">{hit._source.title} at {hit._source.employer}</div>
      {#if hit._source.contacts}
        <div class="text-gray-600">{hit._source.contacts.length} Contacts</div>
      {/if}
      <slot></slot>
    </div>
  </div>
{/if}
