<script>
	import { Howl, Howler } from 'howler'
	import { fade } from 'svelte/transition'
	import { sineInOut } from 'svelte/easing'
	import { onMount } from 'svelte'
	// import Cookies from 'js-cookie'
	import Event from './Event.svelte'
	import CohortClientSession from './CHClientSession.js'
	import Slider from './Slider.svelte'
	import WebsocketConnectionIndicator from './WebsocketConnectionIndicator.svelte'
	import queryString from 'query-string'
	
	
	/*
	 *    Prepare Cohort functionality (for live cues)
	 */	
	let environment = "prod" // can be local, dev, prod
	let cohortSocketURL

	switch(environment){
		case "local":
			cohortSocketURL = 'ws://localhost:3000/sockets'
			break
		case "dev": 
			cohortSocketURL = 'ws://jakemoves-old.local:3000/sockets'
			break
		case "prod":
			cohortSocketURL = 'wss://otm.cohort.rocks/sockets'
			break
		default:
			throw new Error("invalid 'environment' value")
	}

	let itineraryEvent = {
		eventId: 10,
		episodes: [{
			label: "default",
			description: "",
			number: 1,
			cues: []
		}]
	}

	let alertSound = new Howl({
    src: './sounds/ding2.mp3'
  })

	
	let playerLabel = "", playerSleepHours = null, playerActivity = ""
	let didSubmitPlayerInfoForm = false
	let localClearButtonTimeout

	const onSubmitPlayerInfoForm = function(){
		didSubmitPlayerInfoForm = true
		alertSound.play()
		startCohort()
	}
	
	$: sliderBroadcastStatus = "unsent"

	let cohortOccasion = 14
	let connectedToCohortServer = false
	let connectionState = "unknown"
  $: {
    if(connectedToCohortServer === undefined){ connectionState = "unknown" }
    else if(connectedToCohortServer == true){ connectionState = "active" }
    else if(connectedToCohortServer == false){ connectionState = "inactive"}
	}
	
	let selectedOption = ""
	$: cueForSelectedOption = {
		mediaDomain: 3,
		cueNumber: 2,
		cueAction: 0,
		targetTags: ["all"],
		cueContent: selectedOption
	}

	// on text cue
	let latestTextCueContent = ""
	$: splitTextCueContent = latestTextCueContent.split("|")
	let optionButtonLabels
	$: if(splitTextCueContent[0] != ""){
		optionButtonLabels = splitTextCueContent
	} else {
		optionButtonLabels = []
	}

	// on lighting cue
	$: backgroundColor = "rgb(255, 255, 255)"
	let bodyEl = document.getElementsByTagName("body")[0]

	// $: {
	// 	let body = document.getElementsByTagName("body")[0]
	// 	body.setAttribute("style", "background-color: " + backgroundColor)
	// }
	
	let cohortTags, cohortSession, clientPingInterval, connectedOnce

	const startCohort = function(){
		// get grouping info (tags) from URL
		// this is used to target cues to specific groupings
		cohortTags = [ "all" ]
		const parsedQueryString = queryString.parse(location.search)
		// console.log(parsedQueryString)
		const grouping = parsedQueryString.grouping
		// console.log(grouping)
		if(grouping != null && grouping !== undefined){
			cohortTags.push(grouping)
		}

	 	cohortSession = new CohortClientSession(cohortSocketURL, cohortOccasion, cohortTags, playerLabel, playerSleepHours, playerActivity)

		connectedOnce = false
		cohortSession.on('connected', () => {
			connectedToCohortServer = true
			console.log('connected to cohort server')
			connectedOnce = true
			showReconnectButton = false
			clientPingInterval = setInterval(function(){
				const payload = { action: "client_ping", clientGuid: cohortSession.guid }
				cohortSession.send(payload)
				connectedToCohortServer = false
			}, 10000)
		})

		cohortSession.on('disconnected', (message) => {
			connectedToCohortServer = false
			console.log(connectedToCohortServer)
			clearInterval(clientPingInterval)
			latestTextCueContent = ""
			selectedOption = "" // starts to look like a reset function
		})
		cohortSession.on('dataReceived', data => {
			if(data.dataIdentifier == "client_pong"){
				console.log("client-initiated ping/pong successful")
				connectedToCohortServer = true
			}
		})
		cohortSession.on('cueReceived', async (cue) => {
			console.log('cue received:')
			console.log(cue)

			let cueMatchesTarget = false
			
			console.log(cohortTags)
			for(var i = 0; i < cue.targetTags.length; i++){
				console.log(cue.targetTags[i])
				if(cohortTags.includes(cue.targetTags[i])){
					cueMatchesTarget = true
					break
				}
			}

			if(cueMatchesTarget){
				if(cue.mediaDomain == 3 && cue.cueContent !== undefined){
					if(localClearButtonTimeout !== undefined){
						clearTimeout(localClearButtonTimeout)
					}					
					
					if(cue.cueNumber == 1){
						alertSound.play()
						bodyEl.classList.remove('dark')
						latestTextCueContent = cue.cueContent
						await delay(1000)
						bodyEl.classList.add('dark')
						localClearButtonTimeout = setTimeout(() => { 
							if(latestTextCueContent != ""){ latestTextCueContent = "" }
						}, 40000) // hides buttons in case this player misses that cue
					} else if(cue.cueNumber == 2){
						latestTextCueContent = ""
						selectedOption = ""
						if(tellWasChosen == true){
							showTellInstructions = true
						}
					}
				} else if(cue.mediaDomain == 4 && cue.cueContent !== undefined){
					backgroundColor = cue.cueContent
				}
			}
		})

		cohortSession.init()
	}
	
	const onReconnect = async function(){
		try {
      cohortSession.socket = await cohortSession.connect()
    }
    catch( error ) {
			return reject(error) 
    }
	}

	let showReconnectButton = false
	$: (async () => {
		if(connectedOnce && connectionState == "inactive"){
			await delay(1000)
			if(connectionState == "inactive"){
				showReconnectButton = true
			} else {
				showReconnectButton = false
			}
		}
	})()

	let tellWasChosen = false
	let showTellInstructions = false
	const onOptionSelected = function(option){
		selectedOption = option
      // fetch("https://new.cohort.rocks/api/v2/occasions/" + cohortOccasion + "/broadcast", {
      //   method: 'POST',
      //   headers: { 'Content-Type': 'application/json', 'Authorization': 'JWT eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6ImRldl91c2VyIiwiaWF0IjoxNTgzNjExNzk2fQ.k_9oasZ-c3-gvMKOJAHcN9Q56cKkhdeJiU2DlKhCuc4'},
      //   body: JSON.stringify({
			// 		mediaDomain: 3,
			// 		cueNumber: 1,
			// 		cueAction: 0,
			// 		targetTags: ["stage_manager"],
			// 		cueContent: option
			// 	})
			// })
	}

	let showBroadcastSuccess = false
	const onBroadcastResult = function(event){
		const msg = event.detail
		if(msg.broadcastStatus !== undefined && (msg.broadcastStatus == "full-success" || msg.broadcastStatus == "partial-success")){
			latestTextCueContent = ""
			console.log(selectedOption)
			if(selectedOption == "Tell"){
				tellWasChosen = true
			} else { tellWasChosen = false }

			showRadioSwitch = true

			selectedOption = ""
			showBroadcastSuccess = true
			setTimeout(() => { showBroadcastSuccess = false}, 5000)
		}
	}

	let showRadioSwitch = false
	let disableRadioSwitch = false
	let radioOn = false
	const onRadioSwitch = function(event){
		console.log("User set radioOn to " + radioOn)
		disableRadioSwitch = true
		setTimeout( () => { disableRadioSwitch = false }, 30000)
		const radioCue = {
			mediaDomain: 0,
			cueNumber: 1,
			cueAction: radioOn ? 0 : 3,
			targetTags: ["stage_manager"]
		}
		try {
			fetch("https://otm.cohort.rocks/api/v2" + "/occasions/" + 14 + "/broadcast", {
				method: 'POST',
				headers: { 'Content-Type': 'application/json', 'Authorization': 'JWT eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6Iml0aW5lcmFyeV9wbGF5ZXIiLCJpYXQiOjE1OTc5NjgzMjV9._jhynu_UZMCwm0z759twx726_G2J1cVt2tUkNHPVQ6c'},
				body: JSON.stringify(radioCue)
			})
		}
		catch (error){
			console.log(error)
		}
	}
	

	// const onRadioSwitch = function(){
	// 	disableRadioSwitch = true
	// 	setTimeout(() => { disableRadioSwitch = false }, 15000)
	// }

	const delay = function(time){ // time in ms
		return new Promise( resolve => setTimeout(resolve, time))
	}

	/*
	 *    End Cohort
	 */
</script>

<style>	
	ul {
		margin-top: 2rem;
		padding-left: 0;
		width: 100%
	}

	ul > li {
		list-style-type: none;
		height: 4rem;
	}

	button {
		margin-right: 1rem;
		margin-bottom: 1rem;
	}
/* 
	.btn-reconnect-active {
		visibility: hidden;
		opacity: 0;
		transition: opacity 1s linear 1s;
	}

	.btn-reconnect-inactive {
		visibility: visible;
		opacity: 1;
	} */

	.tell-message .close-button {
		position: absolute;
		top: -30px;
		right: 0;
		text-align: right;
	}
</style>

<svelte:head>
	<link rel="stylesheet" href="bootstrap/bootstrap.css">
	<style>
		body {
			/* transition: background-color 5s ease-in-out; */
			background-color: rgb(255, 255, 255);
			transition: background-color 0.5s ease-out;
		}
		body.dark {
			background-color: rgb(0,0,0);
			transition: background-color 30s linear;
		}
		
	</style>
</svelte:head>

<div class="container">
	<div class="row">
		<div class="col">
			{#if !didSubmitPlayerInfoForm}
				<form>
					<div class="form-group">
						<label for="playerLabel">Tell us your first name:</label>
						<input type="text" id="playerLabel" name="playerLabel" bind:value={playerLabel}>
					</div>	
					<div class="form-group">
						<label for="playerLabel">How many hours of sleep did you get last night?</label>
						<input type="number" min="0" step="1" id="playerSleepHours" name="playerSleepHours" bind:value={playerSleepHours}>
					</div>	
					<div class="form-group">
						<label for="playerActivity">What's an activity that gives you calm and contentment?</label>
						<input type="text" id="playerActivity" name="playerActivity" bind:value={playerActivity}>
					</div>
					<button class="btn btn-dark" disabled={ playerLabel == "" || playerSleepHours == null || playerActivity == ""} on:click={ onSubmitPlayerInfoForm }>Submit</button>
				</form>
			{:else}
				<p>
					Player: { playerLabel } <br/>
					Connection: <WebsocketConnectionIndicator status={connectionState}/>
					{#if showReconnectButton}
						<button class={"btn btn-link btn-reconnect-" + connectionState} on:click={onReconnect}>Reconnect</button>
					{/if}
				</p>
			{/if}
		</div>
	</div>

	{#if showRadioSwitch}
		<div class="row" transition:fade={{duration: 500}}>
			<div class="col">
				<div class="form-group">
					Radio: 
					<button class="btn btn-secondary" type="button" on:click={ (e) => {
						radioOn = true
						onRadioSwitch(e)
					}} disabled={ disableRadioSwitch }>Turn on</button>

					<button class="btn btn-secondary" type="button" on:click={ (e) => {
							radioOn = false
							onRadioSwitch(e)
						}} disabled={ disableRadioSwitch }>Turn off</button>
				</div>
			</div>
		</div>
	{/if}

	{#if optionButtonLabels.includes("Tell")}
		<div class="row">
			<div class="col">
				<p>Dear Lucid Dreamers,<br/>
					In the waking world, the patient you are with right now will be taken off life support at 0600.<br/>
					You can decide whether or not to tell them.
				</p>
			</div>
		</div>
	{/if}
	
	<div class="row">
		<div class="col">
		<!-- <Event 
			event={itineraryEvent} 
			connectedToCohortServer={connectedToCohortServer}
			episodeNumberToPlay={episodeNumberToPlay}
		/> -->
		
		<!-- itinerary UI goes here -->
		<!-- <ul class="d-flex flex-wrap mt-8"> -->
			<!-- <li class="mb-4"> -->
			{#each optionButtonLabels as buttonLabel, index}
				<button 
					type="button" 
					class="btn btn-dark text-center"
					transition:fade={{duration: 500, delay: index * 75, easing: sineInOut}}
					on:click={e => onOptionSelected(e.target.innerHTML)}>
					{buttonLabel}
				</button>
			{/each}
			<!-- </li> -->
		<!-- </ul> -->
		</div>
	</div>

	<!-- {#if showTellInstructions }
		<div class="row">
			<div class="col">
				<p class="tell-message">
					Please unmute your mic on Zoom and tell the patient: <br/>
					"I thought you should know, at 6:00 AM, your life will come to an end."
					<button class="btn btn-link close-button" on:click={e => {
						tellWasChosen = false 
						showTellInstructions = false
					}}>[close this message]</button>
				</p>
			</div>
		</div>
	{/if} -->

	<div class="row">
		<div class="col-md-12">
			{#if selectedOption != ""}
				<Slider serverURL={"https://otm.cohort.rocks/api/v2"} focusedOccasionID={14} broadcastStatus={sliderBroadcastStatus} sliderCue={cueForSelectedOption} on:message={onBroadcastResult}></Slider>
			{/if}

			{#if showBroadcastSuccess}
				<p in:fade={{duration: 100}} out:fade={{duration: 1000}}>Your choice was submitted</p>
			{/if}

			<!-- <div class="row mt-4">
				<div class="col text-center">
				{#if cohortSession !== undefined}
					<p class="small">groups: {#each cohortSession.tags as tag}<span class="grouping">{tag}</span>, {/each}</p>
				{/if}
				</div>
			</div> -->
		</div>
	</div>
</div>