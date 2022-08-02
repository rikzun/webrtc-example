<script lang="ts">
    export let stream: MediaStream | null = null
    export let text: string

    let video: HTMLVideoElement
    $: if (stream && video) video.srcObject = stream
</script>

<template>
    {#if stream}
        <div class="video-container">
            <video
                autoplay
                playsinline
                class="video-block"
                bind:this={video}
            />
            <div class="overlay">
                {text}
            </div>
        </div>
    {/if}
</template>

<style lang="sass">
  .video-container
    position: relative

    > .video-block
      display: block
      aspect-ratio: 16 / 9
      height: inherit
      width: 448px

    > .overlay
      left: 0
      bottom: 0
      position: absolute
      background-color: #00000094
      padding-inline: 8px
      color: white
</style>