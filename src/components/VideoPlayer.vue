<template>
  <video
    v-if="shouldRenderVideo"
    ref="videoElement"
    :class="videoClass"
    :width="width"
    :height="height"
    muted
    autoplay
    loop
    :poster="posterUrl"
    @loadstart="setupVideo"
    @error="onVideoError"
    @canplay="onVideoCanPlay"
    @loadeddata="onVideoLoadedData"
  />
  <!-- Fallback poster image for lower priority instances -->
  <img
    v-else-if="shouldShowVideo"
    :class="videoClass"
    :width="width"
    :height="height"
    :src="posterUrl || player.current_media?.image_url"
    style="object-fit: cover;"
  />
</template>

<script setup lang="ts">
import { ref, computed, watch, nextTick, onUnmounted } from "vue";
import { store } from "@/plugins/store";
import { Player, PlayerState } from "@/plugins/api/interfaces";
import Hls from "hls.js";

// Props interface
export interface Props {
  player: Player;
  width?: string | number;
  height?: string | number;
  videoClass?: string;
  posterUrl?: string;
  priority?: number; // 追加: レンダリング優先度
}

const props = withDefaults(defineProps<Props>(), {
  width: "55",
  height: "55",
  videoClass: "video-player",
  posterUrl: "",
  priority: 1, // デフォルト優先度
});

// Video element reference
const videoElement = ref<HTMLVideoElement | null>(null);
let hls: Hls | null = null;
let isSettingUpVideo = false;

// Computed properties
const shouldShowVideo = computed(() => {
  const hasVideoUrl = props.player.current_media?.custom_data?.video_url;
  const isPowered = props.player.powered !== false;
  
  console.log('VideoPlayer: shouldShowVideo computed - powered:', isPowered);
  console.log('VideoPlayer: shouldShowVideo current_media:', props.player.current_media);
  console.log('VideoPlayer: shouldShowVideo hasVideoUrl:', !!hasVideoUrl);
  console.log('VideoPlayer: shouldShowVideo videoUrl:', hasVideoUrl);
  
  return (
    isPowered &&
    hasVideoUrl
  );
});

// Check if this instance should actually render video (highest priority wins)
const shouldRenderVideo = computed(() => {
  if (!shouldShowVideo.value) return false;
  
  // Only active player should potentially render
  if (props.player.player_id !== store.activePlayerId) return false;
  
  // Simple priority check: フルスクリーン表示時は最高優先度のみ、それ以外は2以上
  if (store.showFullscreenPlayer) {
    // フルスクリーン表示中は priority 3 のみ
    return props.priority === 3;
  } else {
    // 通常表示では priority 2 以上（メインUIが優先）
    return props.priority >= 2;
  }
});

// Setup video player for NicoNico MV support (direct MP4 playback)
const setupVideo = async () => {
  // Only setup if this instance should render video
  if (!shouldRenderVideo.value) {
    console.log('VideoPlayer: setupVideo skipped for priority', props.priority, '(shouldRenderVideo=false)');
    return;
  }
  
  if (isSettingUpVideo) {
    console.log('VideoPlayer: setupVideo already in progress, skipping (priority:', props.priority, ')');
    return;
  }
  
  isSettingUpVideo = true;
  console.log('VideoPlayer: setupVideo called for priority', props.priority, '!');
  
  await nextTick();
  
  if (!videoElement.value) {
    console.log('VideoPlayer: No video element found');
    isSettingUpVideo = false;
    return;
  }
  
  if (!props.player.current_media?.custom_data?.video_url) {
    console.log('VideoPlayer: No video URL found in custom_data');
    isSettingUpVideo = false;
    return;
  }

  const videoUrl = props.player.current_media.custom_data.video_url;
  console.log('VideoPlayer: Setting up video with URL for priority', props.priority, ':', videoUrl);

  // Clean up existing HLS instance
  if (hls) {
    hls.destroy();
    hls = null;
  }

  try {
    // Direct MP4 playback without HLS.js
    if (videoElement.value) {
      videoElement.value.src = videoUrl;
      videoElement.value.load();
      
      // Set up event listeners
      const onLoadedData = () => {
        console.log('VideoPlayer: Video loaded for priority', props.priority, ', starting playback. Player state:', props.player.state);
        
        // Initial sync after video loads
        syncVideoToAudio();
        
        // Clean up function for when video unloads or component unmounts
        const cleanup = () => {
          videoElement.value?.removeEventListener('loadeddata', onLoadedData);
          videoElement.value?.removeEventListener('error', onError);
        };
        
        // Store cleanup function for later use
        if(videoElement.value)
          (videoElement.value as any).__syncCleanup = cleanup;
        
        videoElement.value?.removeEventListener('loadeddata', onLoadedData);
        isSettingUpVideo = false;
      };
      
      const onError = (e: any) => {
        console.error('VideoPlayer: Video error:', e);
        videoElement.value?.removeEventListener('error', onError);
        isSettingUpVideo = false;
      };
      
      videoElement.value.addEventListener('loadeddata', onLoadedData);
      videoElement.value.addEventListener('error', onError);
    }
  } catch (error) {
    console.error('VideoPlayer: Failed to setup video:', error);
    isSettingUpVideo = false;
  }
};

// Sync video playback to audio player
const syncVideoToAudio = () => {
  if (!videoElement.value || !shouldRenderVideo.value) return;
  
  // Use store.activePlayerQueue.elapsed_time like other components do
  const activeQueue = store.activePlayerQueue;
  const currentTime = activeQueue?.elapsed_time || 0;
  
  // Only sync if this player matches the active player
  if (props.player.player_id !== store.activePlayerId) {
    return;
  }
  
  // Use the core sync function with smoothing enabled
  syncVideoTimeOnly(currentTime, true);
  
  // Sync play/pause state
  if (props.player.state === PlayerState.PLAYING && videoElement.value.paused) {
    console.log('VideoPlayer: Starting video playback');
    videoElement.value.play().catch(e => console.log('VideoPlayer: Video play failed:', e));
  } else if (props.player.state !== PlayerState.PLAYING && !videoElement.value.paused) {
    console.log('VideoPlayer: Pausing video playback');
    videoElement.value.pause();
  }
};

// Core video time sync function with smooth adjustment
const syncVideoTimeOnly = (targetTime: number, useSmoothing: boolean = true) => {
  if (!videoElement.value || !shouldRenderVideo.value) return;
  
  // Only sync if this player matches the active player
  if (props.player.player_id !== store.activePlayerId) {
    return;
  }
  /**
   * To absorb discrepancies with audio playback.
   * Consider making this configurable later.
   */
  const adjustedTargetTime = Math.max(0, targetTime -1.5);
  const timeDiff = Math.abs(videoElement.value.currentTime - adjustedTargetTime);

  if (!useSmoothing || timeDiff > 1.0) {
    // Large difference or smoothing disabled: direct seek
    console.log('VideoPlayer: Video time correction (direct seek):', videoElement.value.currentTime.toFixed(3), '->', adjustedTargetTime.toFixed(3), 'diff:', timeDiff.toFixed(3));
    videoElement.value.currentTime = adjustedTargetTime;
    videoElement.value.playbackRate = 1.0; // Reset to normal speed
  } else if (timeDiff > 0.1) {
    // Small difference with smoothing: adjust playback rate for smooth sync
    const isVideoBehind = videoElement.value.currentTime < adjustedTargetTime;
    const targetRate = isVideoBehind ? 1.1 : 0.9; // Speed up or slow down slightly

    console.log('VideoPlayer: Video time adjustment (playback rate):', videoElement.value.currentTime.toFixed(3), 'target:', adjustedTargetTime.toFixed(3), 'diff:', timeDiff.toFixed(3), 'rate:', targetRate);
    videoElement.value.playbackRate = targetRate;
  } else {
    // Very small difference or in sync: use normal speed
    if (videoElement.value.playbackRate !== 1.0) {
      console.log('VideoPlayer: Video in sync, resetting playback rate to 1.0');
      videoElement.value.playbackRate = 1.0;
    }
  }
};

// Event handlers
const onVideoError = (event: Event) => {
  console.log('VideoPlayer: Video error:', event);
};

const onVideoCanPlay = () => {
  console.log('VideoPlayer: Video canplay event fired');
};

const onVideoLoadedData = () => {
  console.log('VideoPlayer: Video loadeddata event fired');
};

// Watchers
// Watch store.activePlayer changes
watch(() => store.activePlayer, (newPlayer, oldPlayer) => {
  console.log('VideoPlayer: Store activePlayer changed:', newPlayer?.player_id);
  console.log('VideoPlayer: Store activePlayer current_media object:', newPlayer?.current_media);
  console.log('VideoPlayer: Store activePlayer current_media type:', typeof newPlayer?.current_media);
  console.log('VideoPlayer: Store activePlayer current_media keys:', newPlayer?.current_media ? Object.keys(newPlayer.current_media) : 'no current_media');
  console.log('VideoPlayer: Store activePlayer video_url:', newPlayer?.current_media?.custom_data?.video_url);
}, { deep: true });

// Watch for active player changes
watch(() => store.activePlayerId, (newPlayerId, oldPlayerId) => {
  console.log('VideoPlayer: Active player changed from', oldPlayerId, 'to', newPlayerId, 'this player:', props.player.player_id);
});

// Watch for player object changes
watch(() => props.player, (newPlayer, oldPlayer) => {
  console.log('VideoPlayer: Props player changed:', newPlayer?.player_id);
  console.log('VideoPlayer: Props player current_media object:', newPlayer?.current_media);
  console.log('VideoPlayer: Props player current_media type:', typeof newPlayer?.current_media);
  console.log('VideoPlayer: Props player current_media keys:', newPlayer?.current_media ? Object.keys(newPlayer.current_media) : 'no current_media');
  console.log('VideoPlayer: Props player video_url:', newPlayer?.current_media?.custom_data?.video_url);
}, { deep: true });

// Watch for changes in current_media
watch(() => props.player.current_media, async (newMedia, oldMedia) => {
  console.log('VideoPlayer: current_media changed, checking for video_url:', newMedia?.custom_data?.video_url);
  
  // Clean up any existing sync interval before setting up new video
  if (videoElement.value && (videoElement.value as any).__syncCleanup) {
    (videoElement.value as any).__syncCleanup();
    delete (videoElement.value as any).__syncCleanup;
  }
  
  // Reset video setup flag when media changes
  isSettingUpVideo = false;
  
  // Clean up existing HLS instance when media changes
  if (hls) {
    hls.destroy();
    hls = null;
  }
  
  // Wait for next tick to ensure DOM is updated
  await nextTick();
  
  // Set up new video if available and should render
  if (newMedia?.custom_data?.video_url && shouldRenderVideo.value && videoElement.value) {
    console.log('VideoPlayer: New media has video, setting up video for priority', props.priority);
    setupVideo();
  }
}, { deep: true });

// Watch for player state changes to sync video
watch(() => props.player.state, (newState) => {
  console.log('VideoPlayer: Player state changed:', newState);
  if (shouldRenderVideo.value && videoElement.value && props.player.current_media?.custom_data?.video_url) {
    syncVideoToAudio();
  }
});

// Watch for store.activePlayerQueue elapsed time changes to sync video position
watch(() => store.activePlayerQueue?.elapsed_time, (newTime) => {
  if (shouldRenderVideo.value && videoElement.value && props.player.current_media?.custom_data?.video_url) {
    // Only sync if this is the active player
    if (props.player.player_id === store.activePlayerId) {
      // Use smooth sync for elapsed time updates
      if (newTime !== undefined && newTime !== null) {
        syncVideoTimeOnly(newTime, true);
      }
    }
  }
});

// Watch for queue changes (track changes in queue context)
watch(() => store.activePlayerQueue?.current_item, async (newItem, oldItem) => {
  // Only proceed if this is the active player
  if (props.player.player_id !== store.activePlayerId) {
    return;
  }
  
  if (newItem && newItem !== oldItem) {
    console.log('VideoPlayer: Queue current_item changed, checking for video setup');
    
    // Clean up any existing sync before setting up new video
    if (videoElement.value && (videoElement.value as any).__syncCleanup) {
      (videoElement.value as any).__syncCleanup();
      delete (videoElement.value as any).__syncCleanup;
    }
    
    // Reset video setup flag
    isSettingUpVideo = false;
    
    // Clean up existing HLS instance
    if (hls) {
      hls.destroy();
      hls = null;
    }
    
    // Wait for DOM and store updates
    await nextTick();
    
    // Check if new track has video and setup if needed
    if (props.player.current_media?.custom_data?.video_url && shouldRenderVideo.value && videoElement.value) {
      console.log('VideoPlayer: Queue item has video, setting up video for priority', props.priority);
      setupVideo();
    }
  }
}, { deep: true });

// Watch for queue item ID changes (more reliable detection of track changes)
watch(() => store.curQueueItem?.queue_item_id, async (newItemId, oldItemId) => {
  // Only proceed if this is the active player
  if (props.player.player_id !== store.activePlayerId) {
    return;
  }
  
  if (newItemId && newItemId !== oldItemId) {
    console.log('VideoPlayer: Queue item ID changed from', oldItemId, 'to', newItemId, '- checking for video setup');
    
    // Clean up any existing sync before setting up new video
    if (videoElement.value && (videoElement.value as any).__syncCleanup) {
      (videoElement.value as any).__syncCleanup();
      delete (videoElement.value as any).__syncCleanup;
    }
    
    // Reset video setup flag
    isSettingUpVideo = false;
    
    // Clean up existing HLS instance
    if (hls) {
      hls.destroy();
      hls = null;
    }
    
    // Wait for DOM and store updates
    await nextTick();
    
    // Give props.player.current_media time to update
    const checkAndSetup = async () => {
      // Use props data as it contains the video URL
      const videoUrl = props.player.current_media?.custom_data?.video_url;
      
      console.log('VideoPlayer: Checking video URL from current_media:', videoUrl, 'media:', props.player.current_media?.name);
      
      if (videoUrl && shouldRenderVideo.value && videoElement.value) {
        console.log('VideoPlayer: New queue item has video, setting up video for priority', props.priority);
        setupVideo();
      } else {
        console.log('VideoPlayer: No video URL found for new queue item, skipping setup');
      }
    };
    
    // Check immediately and also after a short delay to handle timing issues
    checkAndSetup();
    setTimeout(checkAndSetup, 100);
  }
});

// Watch for video element ref changes
watch(videoElement, (newEl) => {
  console.log('VideoPlayer: videoElement ref changed:', newEl, 'priority:', props.priority, 'shouldRender:', shouldRenderVideo.value);
  if (newEl && props.player.current_media?.custom_data?.video_url && shouldRenderVideo.value) {
    console.log('VideoPlayer: Video element exists and video_url available, calling setupVideo manually');
    setupVideo();
  }
}, { immediate: true });

// Watch shouldRenderVideo changes to stop/start video appropriately
watch(shouldRenderVideo, (shouldRender, prevShouldRender) => {
  console.log('VideoPlayer: shouldRenderVideo changed from', prevShouldRender, 'to', shouldRender, 'priority:', props.priority);
  
  if (!shouldRender && videoElement.value) {
    // Stop video playback immediately for lower priority instances
    console.log('VideoPlayer: Stopping video for priority', props.priority);
    videoElement.value.pause();
    videoElement.value.src = '';
    
    // Clean up existing sync
    if ((videoElement.value as any).__syncCleanup) {
      (videoElement.value as any).__syncCleanup();
      delete (videoElement.value as any).__syncCleanup;
    }
  } else if (shouldRender && videoElement.value && props.player.current_media?.custom_data?.video_url) {
    // Start video for highest priority instance
    console.log('VideoPlayer: Starting video for priority', props.priority);
    setupVideo();
  }
});

// Cleanup on unmount
onUnmounted(() => {
  // Clean up any existing sync interval
  if (videoElement.value && (videoElement.value as any).__syncCleanup) {
    (videoElement.value as any).__syncCleanup();
  }
  
  // Clean up HLS instance
  if (hls) {
    hls.destroy();
    hls = null;
  }
});
</script>

<style scoped>
.video-player {
  object-fit: cover;
  /* Hide video controls for cleaner appearance */
}

.video-player::-webkit-media-controls {
  display: none !important;
}

.video-player::-webkit-media-controls-panel {
  display: none !important;
}

.video-player::-webkit-media-controls-play-button {
  display: none !important;
}

.video-player::-webkit-media-controls-start-playback-button {
  display: none !important;
}
</style>
