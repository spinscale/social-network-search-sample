<script>
  import Card from './Card.svelte';
  import Search from './Search.svelte';
	import { onMount } from 'svelte';

  const endpoint = 'http://localhost:9200'
  const indexName = "social-network"
  // this would normally be loaded dynamically via the session...
  const ownId = "alexr"
  const headers = { "Content-type": "application/json; charset=UTF-8" }
 
  let page = 'profile'
  let message = '';
  let profile, contacts;

  async function hashchange() {
		// the poor man's router!
		const path = window.location.hash.slice(1);
    page = path 
    message = ''

    if (page === 'contacts') {
      // load network contacts
      let response = await retrieveContacts(ownId);
      if (response.ok) {
        let json = await response.json();
        contacts = json.hits
        console.log(contacts)
      }
    } else if (page === 'search') {
      // also retrieve the profile, so we can pass it to the search component
      let response = await retrieveProfile(ownId);
      if (response.ok) {
        let json = await response.json();
        profile = json
      }
    } else if (page === 'admin') {
      message = 'THIS IS NOT SECURE, DO NOT RUN IN PRODUCTION, BUT ONLY FOR DEMO PURPOSES. NEVER EXPOSE ELASTICSEARCH DIRECTLY SO IT IS REACHABLE FROM YOUR BROWSER'
    } else {
      // fallback to a default
      page = 'profile'

      let response = await retrieveProfile(ownId);
      if (response.ok) {
        let json = await response.json();
        profile = json
      } else {
        // show error
        if (response.status == 404) {
          message = 'Data is missing. Go to admin and reindex'
        } else {
          message = 'Error: ' + response.text()
        }
      }
    }
	}

	onMount(hashchange);

  async function retrieveProfile(id) {
    const url = endpoint + '/' + indexName + '/_doc/' + ownId;
    return fetch(url, { headers: headers })
  }

  async function retrieveContacts(id) {
    const url = endpoint + '/' + indexName + '/_search';
    return fetch(url, {
      method: 'POST',
      headers: headers,
      body: JSON.stringify(
      {
        size: 1000,
        query: {
          terms: {
            "_id": { index: "social-network", id: "alexr", path: "contacts" }
          }
        },
        sort: [ { "name.last.keyword": { order: "asc" } } ]
      })
    })
  }

  async function reindex() {
    await deleteIndex();
    await createIndex();
    await createPipeline();
    await indexContacts();
  }

  async function deleteIndex() {
    const url = endpoint + '/' + indexName;
    return fetch(url, { method: 'DELETE', headers: headers })
  }

  async function createIndex() {
    const url = endpoint + '/' + indexName;
    return fetch(url, 
      {
        method: 'PUT',
        headers: headers,
        body: JSON.stringify({
          mappings: {
            properties: {
              name: {
                properties: {
                  first: {
                    type: "text",
                    fields: { 
                      search_as_you_type: { type: "search_as_you_type" },
                      keyword: { type: "keyword" }
                    }
                  },
                  last: {
                    type: "text",
                    fields: { 
                      search_as_you_type: { type: "search_as_you_type" },
                      keyword: { type: "keyword" }
                    }
                  },
                  full: {
                    type: "text",
                    fields: { 
                      search_as_you_type: { type: "search_as_you_type" },
                      keyword: { type: "keyword" }
                    }
                  }
                }
              },
              employer: { type: "text" },
              contacts: { type: "keyword" },
              title: { type: "keyword" }
            }
          }
        })
      } 
    )
  }

  async function createPipeline() {
    const url = endpoint + '/_ingest/pipeline/full-name-pipeline';
    return fetch(url,
      {
        method: 'PUT',
        headers: headers,
        body: JSON.stringify({
          processors: [
            {
              script: { source: "ctx.name.full = ctx.name.first + ' ' + ctx.name.last" }
            }
          ]
        })
      }
    )
  }

  // do some bulk loading
  async function indexContacts() {
    const dataset = [
      { "index" : { "_id": "alexr" }},
      { "name": { "first" : "Alexander", "last" : "Reelsen" }, "employer" : "Elastic", "title" : "Community Advocate", "contacts" : [ "philippk", "philipk", "philippl" ], "image": '1' },
      { "index" : { "_id": "philipk" }},
      { "name": { "first" : "Philip", "last" : "Kredible" }, "employer" : "Elastic", "title" : "Team Lead", "image": '2'},
      { "index" : { "_id": "philippl" }},
      { "name": { "first" : "Philipp", "last" : "Laughable" }, "employer" : "FancyWorks", "title" : "Senior Software Engineer", "image": '3' },
      { "index" : { "_id": "philippi" }},
      { "name": { "first" : "Philipp", "last" : "Incredible" }, "employer" : "21st Century Marketing", "title" : "CEO", "image": '4' },
      { "index" : { "_id": "philippb" }},
      { "name": { "first" : "Philipp Jean", "last" : "Blatantly" }, "employer" : "Monsters Inc.", "title" : "CEO", "image": '5' },
      { "index" : { "_id": "felixp" }},
      { "name": { "first" : "Felix", "last" : "Philipp" }, "employer" : "Felixia", "title" : "VP Engineering", "image": '6' },
      { "index" : { "_id": "philippk" }},
      { "name": { "first" : "Philipp", "last" : "Krenn" }, "employer" : "Elastic", "title" : "Community Advocate", "image": '1'}
    ]

    const url = endpoint + '/' + indexName + '/_bulk?refresh&pipeline=full-name-pipeline'
    const data = dataset.map(line => JSON.stringify(line)).join('\n') + '\n'
    return fetch(url, { method: 'PUT', headers: headers, body: data })
  }
</script>

<style>
</style>

<svelte:window on:hashchange={hashchange}/>

<nav class="flex items-center justify-between flex-wrap bg-teal-500 p-6">
  <div class="flex items-center flex-shrink-0 text-white mr-6">
    <span class="font-semibold text-xl tracking-tight">MySocialNetwork</span>
  </div>
  <div class="block lg:hidden">
    <button class="flex items-center px-3 py-2 border rounded text-teal-200 border-teal-400 hover:text-white hover:border-white">
      <svg class="fill-current h-3 w-3" viewBox="0 0 20 20" xmlns="http://www.w3.org/2000/svg"><title>Menu</title><path d="M0 3h20v2H0V3zm0 6h20v2H0V9zm0 6h20v2H0v-2z"/></svg>
    </button>
  </div>
  <div class="w-full block flex-grow lg:flex lg:items-center lg:w-auto">
    <div class="text-sm lg:flex-grow">
      <a href="#profile" class="block mt-4 lg:inline-block lg:mt-0 text-teal-200 hover:text-white mr-4">
        Profile
      </a>
      <a href="#contacts" class="block mt-4 lg:inline-block lg:mt-0 text-teal-200 hover:text-white mr-4">
        Contacts
      </a>
      <a href="#search" class="block mt-4 lg:inline-block lg:mt-0 text-teal-200 hover:text-white mr-4">
        Search
      </a>
    </div>
    <div>
      <a href="#admin" class="inline-block text-sm px-4 py-2 leading-none border rounded text-white border-white hover:border-transparent hover:text-teal-500 hover:bg-white mt-4 lg:mt-0">Admin</a>
    </div>
  </div>
</nav>

<div class="mt-8 ml-8 mr-8">

<h1 class="text-4xl font-bold text-teal-700 capitalize mb-5">{page}</h1>

{#if message}
<div role="alert" class="w-1/2 mt-4 mb-4">
  <div class="border border-red-400 rounded bg-red-100 px-4 py-3 text-red-700">
    <p>{message}</p>
  </div>
</div>
{/if}

{#if page == 'search'}
  <Search endpoint={endpoint} profile={profile} />


{:else if page == 'contacts'}
  {#if contacts && contacts.total.value > 0}
    {#each contacts.hits as contact}
      <Card hit={contact} />
    {/each}
  {/if}



{:else if page == 'admin'}

<div>First, you need to ensure, that Elasticsearch is set up properly by allowing CORS. This requires adding the following settings to your configuration</div>

<pre class="mt-6 mb-6"><code>
http.cors.enabled: true
http.cors.allow-origin: /https?:\/\/localhost(:[0-9]+)?/
</code></pre>

<button class="bg-red-500 hover:bg-red-700 text-white py-2 px-4 rounded" on:click={reindex}>
  Delete & Reindex Data
</button>



{:else}
  {#if profile}
    <Card hit={profile} />
  {/if}
{/if}

</div>
