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

			selectedOption = ""
			showBroadcastSuccess = true
			setTimeout(() => { showBroadcastSuccess = false}, 5000)
		}
	}

	const delay = function(time){ // time in ms
		return new Promise( resolve => setTimeout(resolve, time))
	}

	/*
	 *    End Cohort
	 */


	// let overhearEvent = {
	// 	eventId: 6,
	// 	label: "Overhear",
	// 	subLabel: "<em>Winter 2019</em>",
	// 	eventDescription: "[event description can go here].",
	// 	episodes: [{
	// 		label: "Intro",
	// 		description: "",
	// 		number: 1,
	// 		cues: [{
	// 			cueNumber: 1,
	// 			mediaURL: mediaUrlPrefix + "01_Intro_ambience.mp3"
	// 		}]
	// 	},{
	// 		label: "Benny's Story",
	// 		storyteller: "[no storyteller]",
	// 		thumbnailImageURL: "",
	// 		thumbnailImageA11yText: "",
	// 		locationDescription: "",
	// 		duration: "5:29",
	// 		description: "[no description]",
	// 		contentWarning: "[no content warning]",
	// 		number: 2,
	// 		cues: [{
	// 			cueNumber: 1,
	// 			mediaURL: mediaUrlPrefix + "02_Benny_s_Story.mp3"
	// 		}]
	// 	},{
	// 		label: "Kaylyn's Story",
	// 		storyteller: "[no storyteller]",
	// 		thumbnailImageURL: "",
	// 		thumbnailImageA11yText: "",
	// 		locationDescription: "",
	// 		duration: "5:30",
	// 		description: "[no description]",
	// 		contentWarning: "[no content warning]",
	// 		number: 3,
	// 		cues: [{
	// 			cueNumber: 1,
	// 			mediaURL: mediaUrlPrefix + "03_Kaylyn_s_Story.mp3"
	// 		}]
	// 	},{
	// 		label: "Kokum's Story",
	// 		storyteller: "[no storyteller]",
	// 		thumbnailImageURL: "",
	// 		thumbnailImageA11yText: "",
	// 		locationDescription: "",
	// 		duration: "5:52",
	// 		description: "[no description]",
	// 		contentWarning: "[no content warning]",
	// 		number: 4,
	// 		cues: [{
	// 			cueNumber: 1,
	// 			mediaURL: mediaUrlPrefix + "04_Kokum_s_Story.mp3"
	// 		}]
	// 	},{
	// 		label: "Rachel's Story",
	// 		storyteller: "[no storyteller]",
	// 		thumbnailImageURL: "",
	// 		thumbnailImageA11yText: "",
	// 		locationDescription: "",
	// 		duration: "5:15",
	// 		description: "[no description]",
	// 		contentWarning: "[no content warning]",
	// 		number: 5,
	// 		cues: [{
	// 			cueNumber: 1,
	// 			mediaURL: mediaUrlPrefix + "05_Rachel_s_Story.mp3"
	// 		}]
	// 	},{
	// 		label: "Allison's Story",
	// 		storyteller: "[no storyteller]",
	// 		thumbnailImageURL: "",
	// 		thumbnailImageA11yText: "",
	// 		locationDescription: "",
	// 		duration: "5:15",
	// 		description: "[no description]",
	// 		contentWarning: "[no content warning]",
	// 		number: 6,
	// 		cues: [{
	// 			cueNumber: 1,
	// 			mediaURL: mediaUrlPrefix + "06_Allison_s_Story.mp3"
	// 		}]
	// 	},{
	// 		label: "Sara's Story",
	// 		storyteller: "[no storyteller]",
	// 		thumbnailImageURL: "",
	// 		thumbnailImageA11yText: "",
	// 		locationDescription: "",
	// 		duration: "4:53",
	// 		description: "[no description]",
	// 		contentWarning: "[no content warning]",
	// 		number: 7,
	// 		cues: [{
	// 			cueNumber: 1,
	// 			mediaURL: mediaUrlPrefix + "07_Sara_s_Story_and_curtain_call.mp3"
	// 		}]
	// 	}],
	// 	storytellers: [{
	// 		name: "Braden Butler",
	// 		thumbnailImageURL: "media/images/braden-butler.jpg",
	// 		thumbnailImageA11yText: "a photo of actor Braden Butler",
	// 		bio: "Braden Butler was born and raised in Saskatoon, SK, Canada and moved to Edmonton, AB in the fall of 2017 to pursue a BFA in Acting at the University of Alberta. Recent theatre credits include <em>Wild Abandon</em> (Tough Choice Productions), <em>The Gooseberry</em> (Moplip Theatre/NextFest), <em>Ahunwar: The Devil’s Long Nap</em> (Night Canopy Theatre/2018 Edmonton Fringe), and <em>The Hot Baltimore</em>, The Misfit Vault, An Evening with Anon Chekhov, Midsummer Night’s Dream</em> (University of Alberta). He can be seen next (in Canada!) playing the role of Ken in John Logan’s <em>RED</em> at the 2019 Edmonton Fringe and in his first graduating year production as Buckingham in <em>Richard III</em>. He is thrilled to be a part of the creative ensemble with Overhear and for his work to be shared for the first time overseas. ",
	// 		links: [ 
	// 			"http://paypal.me/bradeninspace",
	// 			"http://instagram.com/bradeninspace"
	// 		]
	// 	},{
	// 		name: "Andrea Folster",
	// 		thumbnailImageURL: "media/images/andrea-folster.jpg",
	// 		thumbnailImageA11yText: "a photo of actor Andrea Folster",
	// 		bio: "Andrea Folster is an emerging Saulteaux artist who has called Saskatoon home since the age of six; born in Winnipeg, she also has roots in the Brokenhead Ojibway Nation. Thanks to her many wonderful professors and the support of family and friends, Andrea holds a Bachelor of Fine Arts in Drama with a concentration in Acting from the University of Saskatchewan. In the fall of 2017, Andrea made her professional debut in Saskatoon and been lucky enough to ‘tread the boards’ in the Canadian Prairies a few times since then. She is both excited and nervous about contributing to this iteration of <em>Overhear</em>—while she occasionally writes, sharing this work with the world in this way is entering into uncharted territory; much gratitude and thanks to those who helped with feedback and support throughout the creation process. Miigwech."
	// 	},{
	// 		name: "Elise Pallagi",
	// 		thumbnailImageURL: "media/images/elise-pallagi.jpg",
	// 		thumbnailImageA11yText: "a photo of actor Elise Pallagi",
	// 		bio: "Elise Pallagi is a spoken word poet, multi-disciplinary performance artist, and a member of both the Saskatoon Indigenous Poets Society and the Saskatoon Poetry Slam Team. Her writing and performance work focuses on her personal experiences and struggles as a transgender woman growing up and living in Treaty 6 territory in the Canadian prairies. Elise is a sought after entertainer who has been a featured artist at many special events, such as The Canadian Festival of Spoken Word, The Nutrien Fringe Festival, Regina’s Word Up, Tonight It’s Poetry, several Pride Festivals, and as both a performer and stage MC for The Ness Creek Music Festival.",
	// 		links: [
	// 			"http://elisepallagi.com",
	// 			"http://instagram.com/kandilonglegz",
	// 			"http://facebook.com/elisepallagispokenword",
	// 			"http://paypal.me/elisepallagi"
	// 		]
	// 	},{
	// 		name: "Shanda Stefanson",
	// 		thumbnailImageURL: "media/images/shanda-stefanson.jpg",
	// 		thumbnailImageA11yText: "a photo of actor Shanda Stefanson",
	// 		bio: "Shanda Stefanson is a poet, spoken word artist, playwright and puppeteer. She has competed nationally representing Saskatoon at both the Canadian Festival of Spoken Word and the Canadian Individual Poetry Slam. Her award-winning play <em>Stalled</em> debuted at the Saskatoon Fringe in 2013. She was a member of the collective that created the spoken word play <em>Our Four Walls</em>, which was nominated for a SATA. Shanda is currently working on a one-woman show dealing with the intersection of sex and grief, and can be found at home most nights playing with her cat when she should be writing. "
	// 	},{
	// 		name: "Amberlin Hsu",
	// 		thumbnailImageURL: "media/images/amberlin-hsu.jpg",
	// 		thumbnailImageA11yText: "a photo of actor Amberlin Hsu",
	// 		bio: "Amberlin Hsu is a SATA-winning producer, designer (lighting, costumes, set), and choreographer based in Tokyo and Saskatoon, Canada. She took dance at NTUA in Taiwan and graduated from the University of Saskatchewan with a BFA in Drama (Design). As co-AD of It’s Not A Box Theatre with Torien Cafferata, her recent devised theatre credits include: <em>pimohtēwak</em> (2019), <em>Overhear</em> (2016-2019), <em>cell</em> (2017), <em>Hypneurosis</em> (2016), and <em>Project O</em> (2015). Recent lighting design credits: <em>The Death of a Salesman</em> (Theatre Naught), <em>Displaced</em> (Ground Cover Theatre), <em>Dominion</em> (GTNT), and <em>Les Liaisons Dangereuses</em> (Theatre Naught); Recent costume design credits: <em>The Death of A Salesman</em> (Theatre Naught), <em>Southern Dandy 75</em> (Otto Helmut Productions), <em>Aiden Flynn Lost his Brother, So He Makes Another</em> (Theatre Howl), <em>Macbeth</em> (Embrace Theatre)."
	// 	}]
	// }
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