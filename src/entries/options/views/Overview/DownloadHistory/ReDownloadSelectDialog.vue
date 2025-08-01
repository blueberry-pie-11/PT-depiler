<script setup lang="ts">
import { ref, shallowRef } from "vue";
import type { CAddTorrentOptions } from "@ptd/downloader";

import { sendMessage } from "@/messages.ts";
import { useResetableRef } from "@/options/directives/useResetableRef.ts";
import type { ITorrentDownloadMetadata } from "@/shared/types.ts";

import SentToDownloaderDialog from "@/options/components/SentToDownloaderDialog.vue";

const showDialog = defineModel<boolean>();
const emit = defineEmits<{
  (e: "reDownloadComplete"): void;
}>();

const { torrentItems } = defineProps<{
  torrentItems: ITorrentDownloadMetadata[];
}>();

type TReDownloadType = "old" | "local" | "downloader";

const { ref: isReDownloading, reset: resetIsReDownloading } = useResetableRef<Record<TReDownloadType, boolean>>(() => ({
  old: false,
  local: false,
  downloader: false,
}));

const disableLocalDownload = ref<boolean>(false);
const showSentToDownloaderDialog = ref<boolean>(false);
const downloadTorrentsRef = shallowRef<ITorrentDownloadMetadata["torrent"][]>([]);

const btnItem: Record<TReDownloadType, { icon: string; color: string; title: string }> = {
  old: { icon: "mdi-reload", color: "indigo", title: "按原下载方式重试" },
  local: { icon: "mdi-content-save", color: "orange", title: "本地下载" },
  downloader: { icon: "mdi-cloud-download", color: "cyan", title: "选择下载服务器" },
};

function submitDownloadFinish(reDownloadType: TReDownloadType) {
  isReDownloading.value[reDownloadType] = false;
  emit("reDownloadComplete");
  showDialog.value = false;
}

function reDownload(reDownloadType: TReDownloadType) {
  isReDownloading.value[reDownloadType] = true;
  if (reDownloadType === "downloader") {
    // 对 downloader 则弹出 SentToDownloaderDialog 进行下一步操作
    downloadTorrentsRef.value = torrentItems.map((x) => x.torrent);
    showSentToDownloaderDialog.value = true;
  } else {
    // 对 old 和 local 直接调用下载方法
    const promises = [];

    for (const history of torrentItems) {
      if (history) {
        const historyTorrent = history.torrent;
        if (reDownloadType === "local" || history.downloaderId === "local") {
          promises.push(sendMessage("downloadTorrentToLocalFile", { torrent: historyTorrent }));
        } else {
          promises.push(
            sendMessage("downloadTorrentToDownloader", {
              torrent: historyTorrent,
              downloaderId: history.downloaderId,
              addTorrentOptions: (history.addTorrentOptions ?? {}) as CAddTorrentOptions,
            }),
          );
        }
      }
    }

    Promise.all(promises).finally(() => {
      submitDownloadFinish(reDownloadType);
    });
  }
}

function dialogEnter() {
  resetIsReDownloading();

  // 如果传入的种子列表中有 magnet 链接，则禁用本地下载按钮
  disableLocalDownload.value = torrentItems.some((t) => t?.torrent?.link?.startsWith("magnet:"));
}
</script>

<template>
  <v-dialog v-model="showDialog" max-width="600" @after-enter="dialogEnter">
    <v-card>
      <v-card-title class="pa-0">
        <v-toolbar color="primary">
          <v-toolbar-title>重新下载 {{ torrentItems.length }} 条记录</v-toolbar-title>
          <template #append>
            <v-btn icon="mdi-close" @click="showDialog = false" />
          </template>
        </v-toolbar>
      </v-card-title>
      <v-card-title>
        <v-list>
          <v-list-item v-for="(value, key) in btnItem" :key="key">
            <v-btn
              :loading="isReDownloading[key]"
              block
              class="justify-start"
              :color="value.color"
              :prepend-icon="value.icon"
              size="x-large"
              variant="tonal"
              @click="reDownload(key)"
            >
              {{ value.title }}
            </v-btn>
          </v-list-item>
        </v-list>
      </v-card-title>
    </v-card>
  </v-dialog>

  <SentToDownloaderDialog
    v-model="showSentToDownloaderDialog"
    :torrent-items="downloadTorrentsRef"
    @cancel="() => (isReDownloading.downloader = false)"
    @done="() => submitDownloadFinish('downloader')"
  />
</template>

<style scoped lang="scss"></style>
