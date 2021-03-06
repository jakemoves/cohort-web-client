<script>
	import queryString from 'query-string'
	import AudioPlayer from './AudioPlayer.svelte'
	import { onMount } from 'svelte'
	import CohortClientSession from './CHClientSession.js'
	import WebsocketConnectionIndicator from './WebsocketConnectionIndicator.svelte'
	
	/*
	 *    Prepare Cohort functionality (for live cues)
	 */	
	//
	//This is here because you can't connect locally to secure sites, or vice versa. A local server is required for testing locally.
	//ws = http, wss = https
	let environment = "staging" // can be local, dev, prod
	let cohortSocketURL

	switch(environment){
		case "local":
			cohortSocketURL = 'ws://localhost:3000/sockets'
			break
		case "dev": 
			cohortSocketURL = 'ws://[INSERT LOCAL IP ADDRESS]:3000/sockets'
			break
		case "staging":
			cohortSocketURL = 'wss://staging.cohort.rocks/sockets'
			break
		case "prod":
			cohortSocketURL = 'wss://cohort.rocks/sockets'
			break
		default:
			throw new Error("invalid 'environment' value")
	}

	let cohortOccasion = 14
	let connectedToCohortServer
	let connectionState = "unknown"
  $: {
    if(connectedToCohortServer === undefined){ connectionState = "unknown" }
    else if(connectedToCohortServer == true){ connectionState = "active" }
    else if(connectedToCohortServer == false){ connectionState = "inactive"}
	}
	
	let cohortTags, cohortSession

	onMount(() => {
		startCohort()
	})


	const startCohort = function(){
		// get grouping info (tags) from URL
		// this is used to target cues to specific groupings
		cohortTags = [ "all" ]
		const parsedQueryString = queryString.parse(location.search)
		const grouping = parsedQueryString.grouping
		if(grouping != null && grouping !== undefined){
			cohortTags.push(grouping)
		}

	 	cohortSession = new CohortClientSession(cohortSocketURL, cohortOccasion, cohortTags)

		cohortSession.on('connected', () => {
			connectedToCohortServer = true
		})

		cohortSession.on('disconnected', (message) => {
			connectedToCohortServer = false
		})

		cohortSession.on('cueReceived', async (cue) => {
			console.log('cue received:')
			console.log(cue)

			// do stuff based on the cue (eventually this can be automated based on a cuelist, like in Unity)
		})

		cohortSession.init()
	}
	
	const onReconnect = async function(){
		cohortSession.reconnect()
	}

	let showReconnectButton = false
	$: (async () => {
		if(cohortSession && cohortSession.connectedOnce == true){
			if(connectionState == "inactive"){
				showReconnectButton = true
			} else {
				showReconnectButton = false
			}
		}
	})()

	const delay = function(time){ // time in ms
		return new Promise( resolve => setTimeout(resolve, time))
	}

	/*
	 *    End Cohort
	 */
</script>

<svelte:head>
	<link rel="stylesheet" href="bootstrap/bootstrap.css">
</svelte:head>

<div class="container">
	<div class="row">
		<div class="col">
			{#if showReconnectButton}
				<button class="btn btn-sm btn-warning" on:click={onReconnect}>Reconnect</button>
			{/if}
			<WebsocketConnectionIndicator status={ connectionState }/>
		</div>
	</div>
</div>

