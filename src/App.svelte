<script>
let audioUrl = './sounds/intro-score.mp3'
let isPaused = true
let isLoaded
let audioPlayer, playButton
let audioPreloadSetting = "auto"
let audioDuration

$: state = isPaused ? "paused" : "playing"
$: isLoaded = (audioDuration !== undefined && !isNaN(audioDuration)) ? true : false


function onBtnPlay() {
  isPaused = false
}

function onBtnPause() {
  isPaused = true
}
</script>

{#if isPaused}
	<button 
		type="button" 
		on:click={onBtnPlay} 
		class="btn btn-outline btn-outline-success btn-block"
		bind:this={playButton}>
		Play
	</button>
{:else}
	<button 
		type="button" 
		on:click={onBtnPause} 
		class="btn btn-outline btn-outline-warning btn-block">
		{#if isLoaded}Pause{:else}Loading{/if}
	</button>
{/if}

<audio 
  id="audio_player" 
  src={audioUrl}
  preload={audioPreloadSetting}
  bind:paused={isPaused}
  bind:this={audioPlayer}
  bind:duration={audioDuration}
></audio>
