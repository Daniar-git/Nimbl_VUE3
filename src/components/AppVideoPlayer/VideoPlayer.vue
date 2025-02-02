<script setup lang="ts">
import { computed, onMounted, ref, watch, type PropType, nextTick } from 'vue';
import { useMediaControls, useToggle } from '@vueuse/core';
import Controls from './VideoPlayerControls';
import {
    IconPlay,
    IconPause,
    IconFullScreenOn,
    IconFullScreenOff,
    IconVolume,
    IconSettingsSolid,
    IconChatGPT,
    IconSubtitles,
    IconVoice,
} from '../icons';
import IconAirPlay from '../icons/IconAirPlay.vue';
import type { AudioTrackType, SoundVolumeType } from '@/types';
import AppChatGPTVideo from '../AppChatGPT/ChatGPTVideo.vue';
import { useCustomFullscreen } from '@/composables/useCustomFullscreen';
import type { UseMediaTextTrackSource } from '@vueuse/core';
import { useChatGPTStore } from '@/stores/chatGPT';

const props = defineProps({
    lite: Boolean,
    videoSrc: {
        type: String,
        default: '/NFTsVideo.mp4',
    },
    subtitles: {
        type: Array as PropType<UseMediaTextTrackSource[]>,
        default: undefined,
    },
    voiceTracks: {
        type: Array as PropType<AudioTrackType[]>,
        default: undefined,
    },
});

const videoEl = ref<HTMLVideoElement>();
const videoWrap = ref<HTMLDivElement | null>(null);
const imgBase64 = {
    selected:
        'data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PScwIDAgMzIgMzInIHdpZHRoPScxLjJlbScgaGVpZ2h0PScxLjJlbScgZmlsbD0nIzhDOThGRicgeG1sbnM9J2h0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnJyA+PHBhdGggZmlsbD0nIzhDOThGRicgZD0nbTEzIDI0bC05LTlsMS40MTQtMS40MTRMMTMgMjEuMTcxTDI2LjU4NiA3LjU4NkwyOCA5TDEzIDI0eicvPjwvc3ZnPg==',
};

/* Video ******************************************************************************************* */
const audioEl = ref<HTMLAudioElement>();

const enableVoice = ref<number | null>(null);
const currentVoiceSrc = computed(() => {
    if (props.voiceTracks && typeof enableVoice.value === 'number') {
        return props.voiceTracks[enableVoice.value].src;
    } else {
        return '';
    }
});

/**
 * Выбираем субтитры под озвучку
 */
const currentSubtitlesId = computed(() => {
    if (props.voiceTracks && typeof enableVoice.value === 'number' && props.subtitles) {
        const lang = props.voiceTracks[enableVoice.value].language;
        return props.subtitles.findIndex((s) => s.srcLang === lang);
    } else {
        return false;
    }
});

const {
    playing: playVoice,
    currentTime: currentTimeVoice,
    volume: volumeVoice,
} = useMediaControls(audioEl, {
    src: {
        src: currentVoiceSrc.value,
        type: 'audio/mpeg',
    },
});

/* Наблюдатель переключения аудио */
watch(currentVoiceSrc, async () => {
    if (audioEl.value) {
        audioEl.value.setAttribute('src', currentVoiceSrc.value);
        await nextTick();
        await audioEl.value.play();
        playVoice.value = playing.value;
        handleChangeTime();

        currentSubtitlesId.value && enableTrack(currentSubtitlesId.value); // если есть субтитры под аудио - вкл
    }
});

/**
 * Функция сопоставляет время видео и аудио
 */
function handleChangeTime() {
    currentTimeVoice.value = currentTime.value;
}
onMounted(() => {
    // Аудио наблюдает за Видео
    watch([playing, waiting], () => {
        playVoice.value = !waiting.value && playing.value; // включаем одновременно
        handleChangeTime();
    });
    handleChangeTime();
});

/* **************************************************************************************** */

/* Full Screen */
const { isFullscreen, toggleFullscreen } = useCustomFullscreen(videoWrap);

// const { width: widthVideo, height: heightVideo } = useElementSize(videoWrap);

const controlsVideo = useMediaControls(videoEl, {
    src: {
        src: props.videoSrc,
        type: 'video/mp4',
    },
    tracks: props.subtitles,
});

const {
    playing,
    buffered,
    currentTime,
    duration,
    waiting,
    volume,
    muted,
    supportsPictureInPicture,
    togglePictureInPicture,
    disableTrack,
    selectedTrack,
    tracks,
    enableTrack,
} = controlsVideo;

/* Computed for duration Video */
const endBuffer = computed(() => (buffered.value.length > 0 ? buffered.value[buffered.value.length - 1][1] : 0));
const formatDuration = (seconds: number) => new Date(1000 * seconds).toISOString().slice(14, 19);

/* Visible Controls and Menu */
const [visibleChatGPT, toggleChatGPT] = useToggle();
const visibleOfMousemove = ref(false);
const mousemoveHandler = () => {
    if (!visibleOfMousemove.value) {
        visibleOfMousemove.value = true;
        setTimeout(() => {
            visibleOfMousemove.value = false;
        }, 4000);
    }
    return;
};

const visibleControls = computed(() => {
    return visibleOfMousemove.value || visibleChatGPT.value;
});

/* Звук видео */
const differentBackgroundVolume = 0.6;
const scrubberVolume = ref(0.5);
watch([scrubberVolume, currentVoiceSrc], () => {
    if (currentVoiceSrc.value) {
        volumeVoice.value = scrubberVolume.value;
        if (scrubberVolume.value === 0) {
            volume.value = 0;
        } else if (scrubberVolume.value <= differentBackgroundVolume) {
            volume.value = 0.1;
        } else {
            volume.value = scrubberVolume.value - differentBackgroundVolume;
        }
    } else {
        volume.value = scrubberVolume.value;
    }
});

/* Переключение выкл и вкл звука */
watch(muted, () => {
    if (muted.value) {
        scrubberVolume.value = 0;
    } else if (scrubberVolume.value === 0) {
        scrubberVolume.value = 0.5;
    }
});

/* Computed Volume Icon */
const soundVolume = computed<SoundVolumeType>(() => {
    if (volume.value === 0 || muted.value) {
        return 'off';
    } else if (volume.value > 0.5) {
        return 'high';
    } else if (volume.value < 0.2) {
        return 'low';
    } else {
        return 'medium';
    }
});

// Наблюдение за перемоткой из NimblAI
const GPTStore = useChatGPTStore();
watch(
    () => GPTStore.changeCurrentTimeVideoFromAI,
    (t) => {
        if (t && t > 0) {
            currentTime.value = t;
            GPTStore.changeCurrentTimeVideoFromAI = null
        }
    }
);

/* Change initial media properties */
/* onMounted(() => {
    volume.value = 0.5;
}); */
</script>

<template>
    <div class="flex justify-center duration-200" ref="videoWrap">
        <div
            class="group/video flex-grow outline-none duration-200"
            :tabindex="0"
            autofocus
            @keydown.prevent.stop.space="playing = !playing"
            @keydown.right="currentTime += 10"
            @keydown.left="currentTime -= 10"
            @mousemove="mousemoveHandler">
            <div class="relative h-full w-full overflow-hidden shadow">
                <!-- crossorigin="anonymous" -->
                <video
                    :autoplay="!lite"
                    ref="videoEl"
                    class="block h-full w-full bg-transparent"
                    @click="playing = !playing" />
                <audio ref="audioEl"></audio>
                <div
                    v-if="waiting"
                    class="pointer-events-none absolute inset-0 grid place-items-center bg-black bg-opacity-20">
                    <Controls.Spinner />
                </div>

                <!-- Controls -->
                <div
                    :class="[
                        'absolute inset-x-0 bottom-0 z-50 flex h-1/2 flex-col justify-end !bg-[linear-gradient(0deg,rgba(0,0,0,0.8)_0%,rgba(0,0,0,0.2)_63%,rgba(0,0,0,0)_100%)]',
                        !playing || visibleControls ? 'visible' : 'invisible',
                    ]">
                    <!-- заполнитель пространства  -->
                    <div class="flex-grow" @click="playing = !playing"></div>
                    <!--  -->

                    <!-- Линия перемотки -->
                    <Controls.Scrubber
                        v-model="currentTime"
                        :max="duration"
                        :secondary="endBuffer"
                        class="flex-shrink-0"
                        @click="handleChangeTime">
                        <template #default="{ position, pendingValue }">
                            <div
                                class="absolute bottom-0 mb-4 -translate-x-1/2 transform rounded bg-black px-2 py-1 text-xs text-white"
                                :style="{ left: position }">
                                {{ formatDuration(pendingValue) }}
                            </div>
                        </template>
                    </Controls.Scrubber>
                    <!-- --- -->

                    <!-- Строка Контролеров -->
                    <div :class="['mt-1 flex  flex-row items-center', isFullscreen ? 'px-4 py-3' : 'h-10 px-2 py-1']">
                        <!-- toggle Play -->
                        <button @click="playing = !playing">
                            <IconPlay v-if="!playing" class="inline-block h-8 w-8 align-middle" />
                            <IconPause v-else class="inline-block h-8 w-8 align-middle" />
                        </button>
                        <!--  -->

                        <!-- toggle Muted -->
                        <button @click="muted = !muted" title="Volume" class="ml-4">
                            <IconVolume class="inline-block h-8 w-8 align-middle" :volume="soundVolume" />
                        </button>
                        <!--  -->

                        <!-- Линия звука -->
                        <Controls.Scrubber v-model="scrubberVolume" :max="1" class="ml-2 w-24" />
                        <!-- --- -->

                        <!--Время видео  -->
                        <div class="ml-2 flex flex-1 flex-col text-sm">
                            {{ formatDuration(currentTime) }} / {{ formatDuration(duration) }}
                        </div>
                        <!-- ---  -->

                        <!-- toggle ChatGPT -->
                        <button
                            class="flex items-center justify-center rounded-sm bg-[#4373f8] p-[2px]"
                            title="ChatGPT"
                            @click="toggleChatGPT()"
                            v-if="isFullscreen">
                            <!-- <IconChatGPT class="inline-block h-6 w-6 align-middle text-black" /> -->
                            <div class="inline-block h-6 w-6 align-middle text-black bg-[url('/logo-sm.png')] bg-cover" />
                        </button>
                        <!-- --- -->

                        <!-- toggle Picture In Picture -->
                        <button
                            @click="togglePictureInPicture"
                            v-if="supportsPictureInPicture"
                            title="Picture In Picture">
                            <IconAirPlay class="ml-4 inline-block h-8 w-8 align-middle" />
                        </button>
                        <!-- --- -->

                        <Controls.Menu v-if="subtitles" class="ml-4">
                            <template #default="{ open }">
                                <button @click="open">
                                    <IconSubtitles class="inline-block h-8 w-8 align-middle" />
                                </button>
                            </template>
                            <template #menu="{ close }">
                                <div class="absolute bottom-0 right-0 rounded bg-black py-2 shadow">
                                    <Controls.MenuItem
                                        @keydown.stop.prevent.enter.space="disableTrack()"
                                        @click="
                                            () => {
                                                disableTrack();
                                                close();
                                            }
                                        ">
                                        <span class="flex-1">Off</span>
                                        <img
                                            :src="imgBase64.selected"
                                            alt=""
                                            :class="{ 'opacity-0': selectedTrack !== -1, 'mx-4 h-5 w-5': true }" />
                                    </Controls.MenuItem>
                                    <Controls.MenuItem
                                        v-for="track in tracks"
                                        :key="track.id"
                                        @keydown.stop.prevent.enter.space="enableTrack(track)"
                                        @click="
                                            () => {
                                                enableTrack(track);
                                                close();
                                            }
                                        ">
                                        <span class="flex-1">{{ track.label }}</span>
                                        <img
                                            :src="imgBase64.selected"
                                            alt=""
                                            :class="{ 'opacity-0': track.mode !== 'showing', 'mx-4 h-5 w-5': true }" />
                                    </Controls.MenuItem>
                                </div>
                            </template>
                        </Controls.Menu>

                        <!-- Voice -->
                        <Controls.Menu v-if="voiceTracks" class="ml-4">
                            <template #default="{ open }">
                                <button @click="open">
                                    <IconVoice class="inline-block h-8 w-8 align-middle" />
                                </button>
                            </template>
                            <template #menu="{ close }">
                                <div class="absolute bottom-0 right-0 rounded bg-black py-2 shadow">
                                    <Controls.MenuItem
                                        @click="
                                            () => {
                                                enableVoice = null;
                                                close();
                                            }
                                        ">
                                        <span class="flex-1">Off</span>
                                        <img
                                            :src="imgBase64.selected"
                                            alt=""
                                            :class="{ 'opacity-0': currentVoiceSrc, 'mx-4 h-5 w-5': true }" />
                                    </Controls.MenuItem>
                                    <Controls.MenuItem
                                        v-for="(v, idx) of voiceTracks"
                                        :key="v.language"
                                        @click="
                                            () => {
                                                enableVoice = idx;
                                                close();
                                            }
                                        ">
                                        <img :src="v.flag" alt="" width="28" height="21" class="flex-shrink-0" />
                                        <span class="flex-shrink-0">{{ v.label }}</span>
                                        <img
                                            :src="imgBase64.selected"
                                            alt=""
                                            :class="{
                                                'opacity-0': idx !== enableVoice,
                                                ' mx-10 h-5 w-5 flex-shrink-0': true,
                                            }" />
                                    </Controls.MenuItem>
                                </div>
                            </template>
                        </Controls.Menu>
                        <!-- --- -->

                        <!-- toggle Settings -->
                        <button title="Settings" class="ml-4">
                            <IconSettingsSolid class="inline-block h-8 w-8 align-middle" />
                        </button>
                        <!-- --- -->

                        <!-- toggle FullScreen toggle  -->
                        <button @click="toggleFullscreen" title="Fullscreen" class="ml-4">
                            <IconFullScreenOn v-if="!isFullscreen" class="inline-block h-8 w-8 align-middle" />
                            <IconFullScreenOff v-else class="inline-block h-8 w-8 align-middle" />
                        </button>
                        <!-- --- -->
                    </div>
                    <!--  -->
                </div>
                <!--  -->

                <!-- Video Title -->
                <div
                    :class="[
                        'absolute inset-x-0 top-0 z-20 flex h-1/3 flex-col  bg-gradient-to-b from-black/70 via-black/30 px-5 pt-5',
                        isFullscreen && (!playing || visibleControls) ? 'visible' : 'invisible',
                    ]">
                    <div class="flex justify-between">
                        <h2 class="max-w-[60%] text-2xl font-semibold">
                            IGNORE THE FUD Binance CZ | AAVE Freezes Lending Markets | Polygon Solana NFT
                        </h2>
                        <div class="flex gap-3">
                            <div class="text-right">
                                <p class="text-2xl">Helen_NFT</p>
                                <p class="text-xl text-[#A9A9B7]">245,511 members</p>
                            </div>
                            <img
                                src="/img/users/11.png"
                                alt="userAvatar"
                                width="64"
                                height="64"
                                class="h-16 w-16 rounded-full" />
                        </div>
                    </div>
                    <!-- заполнитель пространства  -->
                    <div class="flex-grow" @click="playing = !playing"></div>
                    <!--  -->
                </div>
                <!-- --- -->
            </div>
        </div>
        <!-- ChatGPT Sidebar -->
        <Transition
            enter-from-class="!flex-[0_0_0px] !max-w-[0%]"
            enter-active-class="!flex-[111_0_27%] !w-[27%]"
            leave-from-class="flex-[111_0_27%] max-w-[27%]"
            leave-active-class="flex-[0_0_0px] max-w-[0%]"
            ><div
                v-if="visibleChatGPT && isFullscreen"
                class="-mr-[15px] max-w-[27%] flex-[111_0_27%] overflow-hidden duration-200 flex">
                <AppChatGPTVideo /></div
        ></Transition>
        <!--  -->
    </div>

    <!-- <pre class="code-block" lang="yaml">{{ text }}</pre> -->
</template>

<style scoped>
video::-webkit-media-text-track-container {
    overflow: visible !important;
    transform: translateY(-50px) !important;
}
</style>
