<script lang="ts" setup>
import {io} from "socket.io-client";
import {Socket} from "socket.io-client";

interface Body {
  name: string
  text: string
  timestamp: string,
  profile_image_url: string
}

const config = useRuntimeConfig()
const connected = ref(false)
const currentPhase = ref('')
const url = ref("")
const sk = ref<Socket>()
const raws = ref<Body[]>([])
const recents = ref([{
  id: "1MYxNgPQNDOKw"
}, {
  id: "1djxXlQVYdBxZ"
}, {
  id: "1YqGoAAQVRQxv"
}])


const results = ref<{ s: number, r: string }[]>([])
console.log(config.public.socket);
onMounted(() => {
  const socket = io(config.public.socket || "http://localhost:3001");
  sk.value = socket

  socket.on('connect', () => {
    connected.value = socket.connected
  })

  socket.on('disconnect', () => {
    connected.value = socket.connected
  })
})

useHead({
  title: "Twitter Summarizer"
})

const handleEnter = () => {
  const id = url.value.split("/")[url.value.split("/").length - 1]
  record(id)
}

const record = (id: string) => {
  raws.value = []
  results.value = []
  if (id && sk.value) {
    sk.value.emit("send_recording_id", id)
    sk.value.on(`message_${id}`, (message: Body) => {
      raws.value.push(message)
      const objDiv = document.getElementById("chat");
      if (objDiv)
        objDiv.scrollTop = objDiv.scrollHeight;
    })
    sk.value?.on(`record_done_${id}`, () => {
      currentPhase.value = 'summarizing'
    })
    sk.value?.on(`summary_done_${id}`, () => {
      currentPhase.value = 'summary_done'
    })
    sk.value?.on(`summary_${id}`, (text) => {
      results.value.push(text)
    })
    currentPhase.value = 'recording'
  }
}
</script>

<template>
  <div class="h-screen">
    <div class="h-screen max-w-xl mx-auto py-12 flex flex-col">
      <img class="w-1/2" src="/logo.png" alt="">
      <div class="border mt-4 overflow-hidden rounded">
        <div class="relative flex items-center text-xl">
          <input
            v-model="url"
            @keyup.enter="handleEnter"
            type="text" name="search" id="search"
            class="outline-none block w-full border-0 py-3 pr-14 text-gray-900 shadow-sm placeholder:text-gray-400"
            placeholder="https://twitter.com/i/spaces/xxx"
          />
          <div class="absolute inset-y-0 right-0 flex py-1.5 pr-1.5">
            <kbd class="inline-flex items-center rounded border border-gray-200 px-1 font-sans text-xs text-gray-400">
              <span>⌘ Enter</span>
            </kbd>
          </div>
        </div>
        <div
          class="border-t border-gray-100 relative z-10 flex justify-between items-center bg-gray-50 px-4 py-2.5 text-xs text-gray-700">
          <div v-if="currentPhase" class="inline-flex gap-2 items-center relative">
            <div class="relative">
              <div class="relative flex h-3 w-3">
                <div
                  class="animate-ping absolute inline-flex h-full w-full rounded-full bg-sky-400 opacity-75"></div>
                <span class="relative inline-flex rounded-full h-3 w-3 bg-sky-500"></span>
              </div>
            </div>
            <div class="capitalize font-semibold">{{ currentPhase }}</div>
          </div>
          <div v-else>
            Please give an twitter space url
          </div>
          <div class="flex-1 flex flex-wrap justify-end items-center">
            Type <kbd
            class="mx-1 flex h-5 w-5 items-center justify-center rounded border bg-white font-semibold sm:mx-2 border-gray-400 text-gray-900">?</kbd>
            for help.
          </div>
        </div>
      </div>
      <div class="flex-1 relative">
        <div class="absolute inset-0 overflow-auto no-scrollbar py-4" id="chat">
          <ul role="list" class="space-y-6">
            <li
              v-for="(activityItem, activityItemIdx) in raws" :key="activityItemIdx"
              class="relative flex gap-x-4"
            >
              <div
                :class="[activityItemIdx === raws.length - 1 ? 'h-6' : '-bottom-6', 'absolute left-0 top-0 flex w-6 justify-center']">
                <div class="w-px bg-gray-200"/>
              </div>
              <img
                :src="activityItem.profile_image_url"
                alt="" class="relative mt-3 h-6 w-6 flex-none rounded-full bg-gray-50"
              />
              <div class="flex-auto rounded-md p-3 ring-1 ring-inset ring-gray-200">
                <div class="flex justify-between gap-x-4">
                  <div class="py-0.5 text-xs leading-5 text-gray-500">
                    <span class="font-medium text-gray-900">{{ activityItem.name }}</span> commented
                  </div>
                  <time :datetime="activityItem.timestamp" class="flex-none py-0.5 text-xs leading-5 text-gray-500">
                    {{ activityItem.timestamp }}
                  </time>
                </div>
                <p class="text-sm leading-6 text-gray-500">{{ activityItem.text }}</p>
              </div>
            </li>
          </ul>
          <template v-if="results.length">
            <h4 class="font-semibold uppercase text-xs text-gray-500 my-3">Results</h4>
            <ul role="list" class="divide-y divide-gray-100 space-y-2">
              <li v-for="(recent, i) in results" :key="i" class="flex items-center justify-between gap-x-4">
                <div class="text-xs space-y-2">
                  <h4 class="font-semibold uppercase text-xs text-gray-500 my-3">Section #{{ i + 1 }}</h4>
                  <div class="bg-green-50 p-3">
                    <div v-for="x in recent.r.split('\n')">{{ x }}</div>
                  </div>
                </div>
              </li>
            </ul>
          </template>
          <template v-if="!currentPhase">
            <h4 class="font-semibold uppercase text-xs text-gray-500 my-3">Recent</h4>
            <ul role="list" class="divide-y divide-gray-100 space-y-2">
              <li v-for="recent in recents" :key="recent.id" class="flex items-center justify-between gap-x-4">
                <div class="min-w-0">
                  <div class="flex items-start gap-x-3">
                    <p class="text-sm font-semibold leading-6 text-gray-900">{{ recent.id }}</p>
                    <p
                      class="rounded-md whitespace-nowrap mt-0.5 px-1.5 py-0.5 text-xs font-medium ring-1 ring-inset text-green-700 bg-green-50 ring-green-600/20">
                      Complete</p>
                  </div>
                  <div class="mt-1 flex items-center gap-x-2 text-xs leading-5 text-gray-500">
                    <p class="whitespace-nowrap">Due on
                      <time datetime="2023-03-17T00:00Z">March 17, 2023</time>
                    </p>
                    <svg viewBox="0 0 2 2" class="h-0.5 w-0.5 fill-current">
                      <circle cx="1" cy="1" r="1"/>
                    </svg>
                    <p class="truncate">Created by Leslie Alexander</p>
                  </div>
                </div>
                <div class="flex flex-none items-center gap-x-4">
                  <a
                    @click="record(recent.id)"
                    class="cursor-pointer rounded-md bg-white px-2.5 py-1.5 text-sm font-semibold text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 hover:bg-gray-50 sm:block">
                    Replay
                  </a>
                </div>
              </li>
            </ul>
          </template>
          <div v-if="currentPhase" class="mt-6 inline-flex gap-2 items-center relative">
            <div class="relative">
              <div class="relative flex h-3 w-3">
                <div
                  class="animate-ping absolute inline-flex h-full w-full rounded-full bg-sky-400 opacity-75"></div>
                <span class="relative inline-flex rounded-full h-3 w-3 bg-sky-500"></span>
              </div>
            </div>
            <div class="capitalize font-semibold">{{ currentPhase }}</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
