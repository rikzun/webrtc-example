<script lang="ts">
    import VideoBlock from "./components/VideoBlock.svelte"

    const constraints = {audio: false, video: true}
    let peer: RTCPeerConnection | null = null

    let localStream: MediaStream | null = null
    let remoteStream: MediaStream | null = null

    let callInProgress = false
    $: callInProgress = localStream !== null || remoteStream !== null

    const signaling = new BroadcastChannel('webrtc')
    signaling.onmessage = (e) => {
        if (!localStream) return

        switch (e.data.type) {
            case 'offer': handleOffer(e.data); break
            case 'answer': handleAnswer(e.data); break
            case 'candidate': handleCandidate(e.data); break
            case 'ready': {
                startConnection()
                break
            }
            case 'bye': if (peer) hangup(); break
            default: console.log('unhandled', e); break
        }
    }

    async function startCall() {
        localStream = await navigator.mediaDevices.getUserMedia(constraints)
        signaling.postMessage({type: 'ready'})
    }

    async function hangup() {
        if (peer) {
            peer.close()
            peer = null
        }
        if (localStream) {
            for (const v of localStream.getTracks()) {
                v.stop()
            }
            localStream = null
            remoteStream = null
        }
    };

    function createPeerConnection() {
        peer = new RTCPeerConnection()
        peer.onicecandidate = (e) => {
            const message = {
                type: 'candidate',
                candidate: e.candidate?.candidate,
                sdpMid: e.candidate?.sdpMid,
                sdpMLineIndex: e.candidate?.sdpMLineIndex
            }

            signaling.postMessage(message)
        }
        peer.ontrack = (e) => remoteStream = e.streams[0]

        if (localStream) {
            for (const v of localStream.getTracks()) {
                peer.addTrack(v, localStream)
            }
        }
    }

    async function startConnection() {
        await createPeerConnection()
        const offer = await peer!.createOffer()

        signaling.postMessage({type: 'offer', sdp: offer.sdp})
        await peer!.setLocalDescription(offer)
    }

    async function handleOffer(offer) {
        await createPeerConnection()
        await peer!.setRemoteDescription(offer)

        const answer = await peer!.createAnswer()
        signaling.postMessage({type: 'answer', sdp: answer.sdp})

        await peer!.setLocalDescription(answer)
    }

    async function handleAnswer(answer) {
        if (!peer) return
        await peer.setRemoteDescription(answer)
    }

    async function handleCandidate(candidate) {
        if (!peer) return
        await peer.addIceCandidate(candidate.candidate ? candidate : null)
    }

</script>

<template>
    <div class="container">
        {#if callInProgress}
            <div class="videos">
                <VideoBlock text="You" stream={localStream} />
                <VideoBlock text="Not you" stream={remoteStream} />
            </div>
        {/if}
        <div class="controls">
            {#if !callInProgress}
                <button on:click={startCall}>
                    Connect
                </button>
            {:else}
                <button on:click={hangup}>
                    Disconnect
                </button>
            {/if}
        </div>
    </div>
</template>

<style global lang="sass">
  $font: "SF Pro Display", "Arial", sans-serif

  @font-face
    font-family: 'SF Pro Display'
    src: url('../static/SFProDisplay-Regular.ttf')
    font-weight: normal
    font-style: normal

  body
    margin: 0
    width: 100vw
    height: 100vh
    font-family: $font
    background-color: black

    > .container
      width: inherit
      min-height: 100vh

      display: flex
      justify-content: center
      align-items: center
      flex-direction: column

      > .videos
        display: flex
        margin-bottom: 1rem

        > div:last-child
          margin-left: 1rem

      > .controls
        > button
          border: unset
          height: 28px
          font-family: $font
          background-color: #007AFF
          border-radius: 24px
          padding-inline: 10px
          font-size: 16px
          color: white
          cursor: pointer
</style>