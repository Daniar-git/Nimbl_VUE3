<script setup lang="ts">
import AppButton from '@/components/AppButton.vue';
import AppTabList from '@/components/AppTabList.vue';
import AppTabListItem from '@/components/AppTabListItem.vue';
import { IconChatGPT, IconClock, IconComments, IconEye, IconFutures, IconVideoGallery } from '@/components/icons';
import { useVideoStore } from '@/stores/video';
import { randomNumber } from '@/utils';
import { storeToRefs } from 'pinia';
import { computed, onUnmounted, ref } from 'vue';
import { useRoute } from 'vue-router';
import VideoPlayer from '@/components/AppVideoPlayer/VideoPlayer.vue'
import AppChatGPTVideo from '@/components/AppChatGPT/ChatGPTVideo.vue';
import { useChatGPTStore } from '@/stores/chatGPT';
import TheTradeTab from '@/components/TheVideoPage/TheTradeAside.vue';

type SidebarTabsTypes = 'videos' | 'comments' | 'trade' | 'chatGpt';

const activeSidebarTab = ref<SidebarTabsTypes>('videos');

const route = useRoute();
const idParams = computed(() => parseInt(route.params.id as string));

const videoStore = useVideoStore();
const { videoList } = storeToRefs(videoStore);
const video = computed(() => videoStore.getVideoById(idParams.value));

const chatStore = useChatGPTStore();

const views = computed(() => randomNumber(90000, 999999));

onUnmounted(() => {
    chatStore.resetVideoChat()
})
</script>

<template>
    <div class="grid h-full w-full grid-cols-12 gap-5 overflow-hidden pt-5 relative">
        <Transition name="left_fade" mode="out-in">
            <div :key="video?.id" class="video-scrollbar col-span-8 overflow-scroll pl-5 pb-height-navigation flex flex-col">
                <VideoPlayer class="min-h-[65vh] w-full self-center" :video-src="video?.videoSrc" :subtitles="video?.subtitles"  :voice-tracks="video?.audioSrc"/>
                <div class="w-full p-4">
                    <h1 class="mb-4 truncate text-2xl">{{ video?.title }}</h1>
                    <div class="mb-4 flex items-center text-lg">
                        <div class="mr-4">{{video?.id === 13 ? '500 views': '100k views'}}</div>
                        <div class="mr-6">{{video?.id === 13 ? '2 seconds ago': '4 hours ago'}}</div>
                        <div class="flex gap-2 text-base text-[#6BB8FF]">
                            <div class="cursor-pointer bg-[rgba(17,17,17,0.3)] p-1 before:content-['#']">ETH-NFT</div>
                            <div class="cursor-pointer bg-[rgba(17,17,17,0.3)] p-1 before:content-['#']">NFT-Trading</div>
                            <div class="cursor-pointer bg-[rgba(17,17,17,0.3)] p-1 before:content-['#']">Discord-Tips</div>
                        </div>
                    </div>
                    <div class="flex items-center">
                        <div class="inline-flex bg-gradient-tab-list-mute">
                            <div class="inline-flex items-center gap-3 py-3 px-4">
                                <IconVideoGallery class="h-6 w-6" /> <span class="text-lg leading-none">5 304</span>
                            </div>
                            <div class="inline-flex items-center gap-3 py-3 px-4">
                                <IconComments class="h-6 w-6" /> <span class="text-lg leading-none">5 304</span>
                            </div>
                            <div class="inline-flex items-center gap-3 py-3 px-4">
                                <IconFutures class="h-6 w-6" /> <span class="text-lg leading-none">5 304</span>
                            </div>
                        </div>
                        <div class="ml-auto mr-32 text-xl">
                            <AppButton class="mr-5">Subscribe</AppButton> <AppButton @click="$router.push({name: 'channel'})">Channel page</AppButton>
                        </div>
                    </div>
                </div>
            </div>
        </Transition>
        <div class="col-span-4 flex flex-col overflow-hidden">
            <div class="flex pr-5">
                <AppTabList class="h-14 flex-grow !bg-gradient-tab-list-mute dark:!bg-none" @change-tab="(tab) => activeSidebarTab = tab" v-slot="{ onChange }">
                    <AppTabListItem value="videos" :active-value="activeSidebarTab" @click="() => onChange('videos')">
                        <IconVideoGallery class="mr-4 h-5 w-5 translate-y-[2px]" />
                        <span class="text-lg leading-none">Videos</span>
                    </AppTabListItem>
                    <AppTabListItem
                        value="comments"
                        :active-value="activeSidebarTab"
                        @click="() => onChange('comments')">
                        <IconComments class="mr-4 h-5 w-5 translate-y-[3px]" />
                        <span class="text-lg leading-none">Comments</span>
                    </AppTabListItem>
                    <AppTabListItem value="trade" :active-value="activeSidebarTab" @click="() => onChange('trade')">
                        <IconFutures class="mr-4 h-5 w-5" />
                        <span class="text-lg leading-none">Trade</span>
                    </AppTabListItem>
                    <AppTabListItem value="chatGpt" :active-value="activeSidebarTab" @click="() => onChange('chatGpt')">
                        <!-- <IconChatGPT class="h-5 w-5" /> -->
                        <div class="h-8 w-8 bg-[url('/logo-sm.png')] bg-cover" />
                    </AppTabListItem>
                </AppTabList>
            </div>
            <!-- Videos tab -->
            <div v-if="activeSidebarTab === 'videos'" class="mr-[5px] pt-5 overflow-y-scroll pr-[10px] pb-height-navigation">
                <div
                    @click="() => $router.push({name: 'video', params: {id : video.id}})"
                    v-for="video in videoList"
                    :key="video.id"
                    class="mb-4 flex last:mb-0 cursor-pointer hover:bg-gradient-active-tab"
                    aria-orientation="horizontal">
                    <img :src="video.poster" class="aspect-video w-60" alt="poster" />
                    <div class="flex flex-col py-1 pl-5 pr-2" aria-orientation="vertical">
                        <div class="mb-5 line-clamp-2">{{ video.title }}</div>
                        <div class="flex mt-auto items-center gap-2">
                            <img :src="video.avatar" class="block h-6 w-6 rounded-full object-cover" alt="avatar" />
                            <div @click="$router.push({name: 'channel'})" class="text-white/50">{{ video.channel }}</div>
                        </div>
                        <div class="flex items-center gap-6 pt-[5px]">
                            <div class="inline-flex items-center">
                                <IconEye class="mr-2 inline-block w-4 translate-y-[1px]" />
                                <span class="text-xs leading-none">{{ views }}</span>
                            </div>
                            <div class="inline-flex items-center">
                                <IconClock class="mr-2 inline-block h-3 w-3" />
                                <span class="text-xs leading-none">9 days ago</span>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
            <!-- --- -->
            <!-- Trade tab -->
            <div v-if="activeSidebarTab === 'trade'" class="mr-[5px] pr-[10px] pb-height-navigation overflow-y-scroll">
                <TheTradeTab />
            </div>
            <!-- --- -->
            <!-- ChatGPT tab -->
            <AppChatGPTVideo v-if="activeSidebarTab === 'chatGpt'" class="mb-height-navigation" />
    <!-- --- -->
        </div>
    </div>
</template>

<style scoped lang="scss">
.video-scrollbar {
    &::-webkit-scrollbar {
        width: 0px;
    }
    &::-webkit-scrollbar-thumb {
        background: transparent;
    }
}


.left_fade-enter-active,
.left_fade-leave-active {
  transition: all 0.5s ease;
}


.left_fade-leave-to {
  opacity: 0;
  transform: translateY(50%);
}
.left_fade-enter-from {
    transform: translateY(-50%) ;
    opacity: 0;
}

</style>
