<script>
import { onMount } from 'svelte';
import Card from './Card.svelte';
export let endpoint;
export let profile;

// possible fix: export let as profile?
let showQuery = false
let selectedQuery = null;
let selectedQueryAsText = null
let input = '';
let hits = null;

// collect all the queries
const queries = [
  { 
    name: "Simple Query",
    query: { query: { match: { "name.first": "%%INPUT%%" } } }
  },
  {
    name: "Score own contacts higher",
    query: {
      "query": {
        "bool": {
          "should": [
            { "terms": { "_id": { "index": "social-network", "id": "%%OWNID%%", "path": "contacts" } } }
          ],
          "must": [ { "match": { "name.first": "%%INPUT%%" } } ]
        }
      }
    } 
  },
  {
    name: "Score first/last name, score own contacts higher",
    query: {
      "query": {
        "bool": {
          "should": [
            { "terms": { "_id": { "index": "social-network", "id": "%%OWNID%%", "path": "contacts" } } }
          ],
          "must": [ { "multi_match": { "query": "%%INPUT%%", "fields": [ "name.first", "name.last" ] } } ]
        }
      }
    } 
  },
  {
    name: "Score first/last name, score own contacts & colleagues higher",
    query: {
      "query": {
        "bool": {
          "should": [
            { "terms": { "_id": { "index": "social-network", "id": "%%OWNID%%", "path": "contacts" } } },
            { "match": { "employer": "Elastic" } }
          ],
          "must": [ { "multi_match": { "query": "%%INPUT%%", "fields": [ "name.first", "name.last" ] } } ]
        }
      }
    } 
  },
  {
    name: "Search as you type: Score first/last name, score own contacts & colleagues higher",
    query: {
      "query": {
        "bool": {
          "should": [
            { "terms": { "_id": { "index": "social-network", "id": "%%OWNID%%", "path": "contacts" } } },
            { "match": { "employer": "Elastic" } }
          ],
          "must": [
            {
              "multi_match": {
                "query": "%%INPUT%%",
                "boost": 2, 
                "type": "phrase_prefix", 
                "fields": [ "name.last.search_as_you_type", "name.first.search_as_you_type" ]
              }
            } 
          ]
        }
      }
    }
  },
  {
    name: "Search as you type: Score first/last/full name, score own contacts & colleagues higher",
    query: {
      "query": {
        "bool": {
          "should": [
            { "terms": { "_id": { "index": "social-network", "id": "%%OWNID%%", "path": "contacts" } } },
            { "match": { "employer": "Elastic" } }
          ],
          "must": [
            {
              "multi_match": {
                "query": "%%INPUT%%",
                "boost": 2, 
                "type": "phrase_prefix", 
                "fields": [ "name.full.search_as_you_type", "name.last.search_as_you_type", "name.first.search_as_you_type" ]
              }
            } 
          ]
        }
      }
    } 
  },
  {
    name: "Search as you type: Score first/last/full name (ignore term order), score own contacts & colleagues higher",
    query: {
      "query": {
        "bool": {
          "should": [
            { "terms": { "_id": { "index": "social-network", "id": "%%OWNID%%", "path": "contacts" } } },
            { "match": { "employer": "Elastic" } }
          ],
          "must": [
            {
              "bool" : {
                "should": [
                  {
                    "multi_match": {
                      "query": "%%INPUT%%",
                      "boost": 2, 
                      "type": "phrase_prefix", 
                      "fields": [ "name.full.search_as_you_type", "name.last.search_as_you_type", "name.first.search_as_you_type" ]
                    }
                  },
                  {
                    "multi_match": {
                      "query": "%%INPUT%%",
                      "operator": "and",
                      "boost": 2,
                      "type": "bool_prefix",
                      "fields": [ "name.full.search_as_you_type", "name.full.search_as_you_type._2gram", "name.full.search_as_you_type._3gram" ]
                    }
                  }
                ]
              }
            }
          ]
        }
      }
    } 
  }
]

// run the search
async function executeSearch() {
  if (input !== null && input.length > 0) {
    selectedQueryAsText = JSON.stringify(selectedQuery, '', 2)
      .replaceAll('%%INPUT%%', JSON.parse(JSON.stringify(input)))
      .replaceAll('%%OWNID%%', JSON.parse(JSON.stringify(profile._id)))

    const url = endpoint + '/social-network/_search'
    let response = await fetch(url, {
      headers: { "Content-type": "application/json; charset=UTF-8" },
      method: 'POST',
      body: selectedQueryAsText
    })
    if (response.ok) {
      let json = await response.json();
      if (json && json.hits.total.value > 0) {
        hits = json.hits.hits
      } else {
        hits = []
      }
    }
  }
}

// set the first query on mount
onMount(() => {
  selectedQuery = queries[0].query;
})

</script>

<div class="mb-4">
<select bind:value={selectedQuery} class="bg-gray-200 border border-gray-500 text-gray-700 py-1 px-2 mr-4 rounded">
{#each queries as query}
  <option value={query.query}>{query.name}</option>
{/each}
</select>

<button class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-1 px-1 rounded" on:click={() => showQuery = !showQuery}>
  {#if showQuery}
    Hide query
  {:else}
    Show query
  {/if}
</button>
</div>

{#if selectedQueryAsText && showQuery}
<div class="mt-4 font-light text-xs">
  <pre><code>{selectedQueryAsText}</code></pre>
</div>
{/if}

<div class="mt-4">
  <input class="bg-gray-200 py-1 px-1 text-gray-700 focus:border-gray-700 text-xl" type="text" placeholder="Search by name" bind:value={input} on:keyup={executeSearch}>
</div>

{#if hits}
  <div class="mt-4">
    {#each hits as hit}
      <Card hit={hit} border={profile._source.contacts && profile._source.contacts.includes(hit._id) ? 'border-red-700' : 'border-gray-500'}>
        <div class="text-gray-800 text-xs">Score: {hit._score}</div>
      </Card>
    {/each}
  </div>
{/if}
