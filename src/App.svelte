<script>
	import AudioPlayer from './AudioPlayer.svelte'
</script>

<svelte:head>
	<link rel="stylesheet" href="bootstrap/bootstrap.css">
</svelte:head>

<AudioPlayer audioUrl={"." + "/" + 'sounds/intro-score.mp3'}/>