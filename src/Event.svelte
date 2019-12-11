<script>
import { fade, fly } from 'svelte/transition'
import { onMount } from 'svelte'

export let event

export let connectedToCohortServer

// this prop allows cohort server to trigger episode playback
export let episodeNumberToPlay

let showAudioPermissionsModal = false
let audioPlayer
let audioURL
let isFirstPlay = true

let currentEpisode = event.episodes[0]

$: if(episodeNumberToPlay != null){
	if(episodeNumberToPlay != currentEpisode.number){
		playEpisode(episodeNumberToPlay)
	}
}
 
onMount( () => {
	audioURL = currentEpisode.cues[0].mediaURL

	audioPlayer.addEventListener('play', () => {
		console.log('player: play')
	})
	audioPlayer.addEventListener('loadeddata', () => {
		console.log('player: loadeddata')
		if(!isFirstPlay){
			playAudio()
		}
	})
})

function playEpisode(){
	console.log(1)
	if(episodeNumberToPlay == null || episodeNumberToPlay === undefined){
		return
	}

	let episode = event.episodes.find( episode => {
		return episode.number == episodeNumberToPlay
	})

	if(episode === undefined){
		return
	}

	currentEpisode = episode
	audioURL = currentEpisode.cues[0].mediaURL
	console.log(audioURL)

	playAudio()
}

function playAudio(){
	audioPlayer.play()
	.then( () => {
		if(isFirstPlay){
			setTimeout(function(){
				audioPlayer.pause()
			}, 2000)
			isFirstPlay = false
		}
		// if the play() succeeded, hide the permission modal
		// showAudioPermissionsModal = false
	})
	.catch( error => {
		if(
			error.message == "The request is not allowed by the user agent or the platform in the current context, possibly because the user denied permission." ||
			error.message == "The operation was aborted.") {

			// remote play failed because the user hasn't manually started any audio on the page yet
			console.log("need user permission for audio playback")
			// showAudioPermissionsModal = true
		} else {
			console.log(error)
		}
	})
}
</script>

<style>
h1, h2 {
	text-align: center;
}

h1 {
	font-size: 1rem;
	text-transform:uppercase;
	font-variant: small-caps;
}

.event-description {
	text-align: center;
	font-size: 0.9rem;
	line-height: 1.2rem;
}

</style>

<h1>{event.label}
	{#if connectedToCohortServer}
		<span class="text-success">+ live</span>
	{/if}
</h1>
{#if event.thumbnailImageURL}
	<img src={event.thumbnailImageURL} alt="placeholder image" class="img-fluid">
{/if}
{#if event.subLabel}
	<h2>{@html event.subLabel}</h2>
{/if}

{#if event.eventDescription }
<p class="event-description">{event.eventDescription}</p>
{/if}

{#if isFirstPlay}
<button class="btn btn-success" on:click={playEpisode}>
	Join
</button>
{/if}

<audio 
  id="event-player" 
  src={audioURL}
  preload="auto"
  bind:this={audioPlayer}
></audio>

<!-- Modal -->
{#if showAudioPermissionsModal}
<div class="modal" transition:fly={{y: -200, duration: 200}} id={"audioPermissionModalEvent"} tabindex="-1" role="dialog" aria-labelledby={"modalTitleEvent"} aria-hidden="true">
  <div class="modal-dialog modal-dialog-centered" role="document">
    <div class="modal-content">
      <div class="modal-body text-center">
        <p id={"modalTitleEvent"}>
          We need your permission to play audio...
        </p>
        <button class="btn btn-success btn-block mb-4" on:click={playAudio}>
          Tap to allow Overhear Live to play audio
        </button>
      </div>
    </div>
  </div>
</div>
{/if}
