<template>
  <div>
    <breadcrumbs :base="'/share/' + hash" />

    <div v-if="layoutStore.loading">
      <h2 class="message delayed" style="padding-top: 3em !important">
        <div class="spinner">
          <div class="bounce1"></div>
          <div class="bounce2"></div>
          <div class="bounce3"></div>
        </div>
        <span>{{ t("files.loading") }}</span>
      </h2>
    </div>
    <div v-else-if="error">
      <div v-if="error.status === 401">
        <div class="card floating" id="password" style="z-index: 9999999">
          <div v-if="attemptedPasswordLogin" class="share__wrong__password">
            {{ t("login.wrongCredentials") }}
          </div>
          <div class="card-title">
            <h2>{{ t("login.password") }}</h2>
          </div>

          <div class="card-content">
            <input
              v-focus
              class="input input--block"
              type="password"
              :placeholder="t('login.password')"
              v-model="password"
              @keyup.enter="fetchData"
            />
          </div>
          <div class="card-action">
            <button
              class="button button--flat"
              @click="fetchData"
              :aria-label="t('buttons.submit')"
              :data-title="t('buttons.submit')"
            >
              {{ t("buttons.submit") }}
            </button>
          </div>
        </div>
        <div class="overlay" />
      </div>
      <errors v-else :errorCode="error.status" />
    </div>
    <div v-else-if="req !== null">
      <div class="share">
        <div
          class="share__box share__box__info"
          style="position: -webkit-sticky; position: sticky; z-index: 999"
          :style="{
            top: isMobile ? '-25.6em' : '4em',
          }"
        >
          <div class="share__box__header" style="height: 3em">
            {{
              req.isDir
                ? t("download.downloadFolder")
                : t("download.downloadFile")
            }}
          </div>
          <div
            v-if="!req.isDir"
            class="share__box__element share__box__center share__box__icon"
          >
            <img v-if="req.type === 'image'" :src="raw" />
            <i v-else class="material-icons">{{ icon }}</i>
          </div>
          <div class="share__box__element" style="height: 3em">
            <strong>{{ $t("prompts.displayName") }}</strong> {{ req.name }}
          </div>
          <div v-if="!req.isDir" class="share__box__element" :title="modTime">
            <strong>{{ $t("prompts.lastModified") }}:</strong> {{ humanTime }}
          </div>
          <div class="share__box__element" style="height: 3em">
            <strong>{{ $t("prompts.size") }}:</strong> {{ humanSize }}
          </div>
          <div class="share__box__element share__box__center">
            <a
              target="_blank"
              :href="link"
              class="button button--flat"
              style="height: 4em"
            >
              <div>
                <i class="material-icons">file_download</i
                >{{ t("buttons.download") }}
              </div>
            </a>
            <a
              target="_blank"
              :href="inlineLink"
              class="button button--flat"
              v-if="!req.isDir"
            >
              <div>
                <i class="material-icons">open_in_new</i
                >{{ t("buttons.openFile") }}
              </div>
            </a>
          </div>
          <div class="share__box__element share__box__center">
            <qrcode-vue :value="link" :size="200" level="M"></qrcode-vue>
          </div>
          <div
            v-if="req.isDir"
            class="share__box__element share__box__header"
            style="height: 3em"
          >
            {{ $t("sidebar.preview") }}
          </div>
          <div
            v-if="req.isDir"
            class="share__box__element share__box__center share__box__icon"
            style="padding: 0em !important; height: 12em !important"
          >
            <a
              target="_blank"
              :href="raw"
              class="button button--flat"
              v-if="
                !fileStore.multiple &&
                fileStore.selectedCount === 1 &&
                req.items[fileStore.selected[0]].type === 'image'
              "
              style="height: 12em; padding: 0; margin: 0"
            >
              <img style="height: 12em" :src="raw" />
            </a>
            <div
              v-else-if="
                !fileStore.multiple &&
                fileStore.selectedCount === 1 &&
                req.items[fileStore.selected[0]].type === 'audio'
              "
              style="height: 12em; padding-top: 1em; margin: 0"
            >
              <button
                @click="play"
                v-if="!tag"
                style="
                  font-size: 6em !important;
                  border: 0px;
                  outline: none;
                  background: white;
                "
                class="material-icons"
              >
                play_circle_filled
              </button>
              <button
                @click="play"
                v-if="tag"
                style="
                  font-size: 6em !important;
                  border: 0px;
                  outline: none;
                  background: white;
                "
                class="material-icons"
              >
                pause_circle_filled
              </button>
              <audio
                id="myaudio"
                ref="audio"
                :src="raw"
                controls
                :autoplay="tag"
              ></audio>
            </div>
            <video
              v-else-if="
                !fileStore.multiple &&
                fileStore.selectedCount === 1 &&
                req.items[fileStore.selected[0]].type === 'video'
              "
              style="height: 12em; padding: 0; margin: 0"
              :src="raw"
              controls
            >
              Sorry, your browser doesn't support embedded videos, but don't
              worry, you can <a :href="raw">download it</a>
              and watch it with your favorite video player!
            </video>
            <i
              v-else-if="
                !fileStore.multiple &&
                fileStore.selectedCount === 1 &&
                req.items[fileStore.selected[0]].isDir
              "
              class="material-icons"
              >folder
            </i>
            <i v-else class="material-icons">call_to_action</i>
          </div>
        </div>
        <div
          v-if="req.isDir && req.items.length > 0"
          class="share__box share__box__items"
          style="background-color: transparent"
        >
          <FileListing
            :header-sticky-top="isMobile ? '19em' : '4em'"
            :enable-search="false"
            :show-multiple-selection-button="false"
          >
            <action
              v-if="fileStore.selectedCount"
              icon="file_download"
              :label="t('buttons.download')"
              @action="download"
              :counter="fileStore.selectedCount"
            />
            <button
              v-if="isSingleFile()"
              class="action copy-clipboard"
              :aria-label="t('buttons.copyDownloadLinkToClipboard')"
              :data-title="t('buttons.copyDownloadLinkToClipboard')"
              @click="copyToClipboard(linkSelected())"
            >
              <i class="material-icons">content_paste</i>
            </button>
            <action
              icon="check_circle"
              :label="t('buttons.selectMultiple')"
              @action="toggleMultipleSelection"
            />
          </FileListing>
        </div>
        <div
          v-else-if="req.isDir && req.items.length === 0"
          class="share__box share__box__items"
        >
          <h2 class="message">
            <i class="material-icons">sentiment_dissatisfied</i>
            <span>{{ t("files.lonely") }}</span>
          </h2>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { pub as api } from "@/api";
import { filesize } from "@/utils";
import dayjs from "dayjs";

import Action from "@/components/header/Action.vue";
import Breadcrumbs from "@/components/Breadcrumbs.vue";
import Errors from "@/views/Errors.vue";
import QrcodeVue from "qrcode.vue";
import FileListing from "@/components/files/FileListing.vue";

import { useFileStore } from "@/stores/file";
import { useAuthStore } from "@/stores/auth";
import { useLayoutStore } from "@/stores/layout";

import { computed, inject, onMounted, onBeforeUnmount, ref, watch } from "vue";
import { useRoute } from "vue-router";
import { useI18n } from "vue-i18n";
import { StatusError } from "@/api/utils";
import { copy } from "@/utils/clipboard";
import { storeToRefs } from "pinia";
import { useMediaQuery } from "@vueuse/core";

const isMobile = useMediaQuery("(max-width: 1125px)");
const error = ref<StatusError | null>(null);
const showLimit = ref<number>(100);
const password = ref<string>("");
const attemptedPasswordLogin = ref<boolean>(false);
const hash = ref<string>("");
const token = ref<string>("");
const audio = ref<HTMLAudioElement>();
const tag = ref<boolean>(false);

const $showSuccess = inject<IToastSuccess>("$showSuccess")!;

const { t } = useI18n({});

const route = useRoute();
const fileStore = useFileStore();
const { reload } = storeToRefs(fileStore);
const layoutStore = useLayoutStore();
const authStore = useAuthStore();

watch(route, () => {
  showLimit.value = 100;
  fetchData();
});

const req = computed(() => fileStore.req);

// Define computes

const icon = computed(() => {
  if (req.value === null) return "insert_drive_file";
  if (req.value.isDir) return "folder";
  if (req.value.type === "image") return "insert_photo";
  if (req.value.type === "audio") return "volume_up";
  if (req.value.type === "video") return "movie";
  return "insert_drive_file";
});

const link = computed(() => (req.value ? api.getDownloadURL(req.value) : ""));

// 用于转换 URL 的辅助函数
function transformUrl(url: string) {
  return url.replace(/share/, "api/public/dl") + "?token=" + token.value;
}
const raw = computed(() => {
  // 如果 req.value 不存在，则直接返回空字符串
  if (!req.value) {
    return "";
  }

  // 如果 req.value 是目录
  if (req.value.isDir) {
    const selectedItemUrl = req.value.items[fileStore.selected[0]].url;
    return transformUrl(selectedItemUrl);
  }

  // 如果 req.value 不是目录
  return transformUrl(req.value.url);
});

const inlineLink = computed(() =>
  req.value ? api.getDownloadURL(req.value, true) : ""
);
const humanSize = computed(() => {
  if (req.value) {
    return req.value.isDir
      ? req.value.items.length
      : filesize(req.value.size ?? 0);
  } else {
    return "";
  }
});
const humanTime = computed(() => dayjs(req.value?.modified).fromNow());
const modTime = computed(() =>
  req.value
    ? new Date(Date.parse(req.value.modified)).toLocaleString()
    : new Date().toLocaleString()
);

// Functions
const play = () => {
  if (tag.value) {
    audio.value?.pause();
    tag.value = false;
  } else {
    audio.value?.play();
    tag.value = true;
  }
};

watch(reload, (newValue) => {
  newValue && fetchData();
});

const fetchData = async () => {
  fileStore.reload = false;
  fileStore.selected = [];
  fileStore.multiple = false;
  layoutStore.closeHovers();

  // Set loading to true and reset the error.
  layoutStore.loading = true;
  error.value = null;
  if (password.value !== "") {
    attemptedPasswordLogin.value = true;
  }

  let url = route.path;
  if (url === "") url = "/";
  if (url[0] !== "/") url = "/" + url;
  try {
    const file = await api.fetch(url, password.value);
    file.hash = hash.value;

    token.value = file.token || "";
    authStore.guestJwt = token.value;

    fileStore.updateRequest(file);
    document.title = `${file.name} - ${document.title}`;
  } catch (err) {
    if (err instanceof Error) {
      error.value = err;
    }
  } finally {
    layoutStore.loading = false;
  }
};

const keyEvent = (event: KeyboardEvent) => {
  if (event.key === "Escape") {
    // If we're on a listing, unselect all
    // files and folders.
    if (fileStore.selectedCount > 0) {
      fileStore.selected = [];
    }
  }
};

const toggleMultipleSelection = () => {
  fileStore.toggleMultiple();
  layoutStore.closeHovers();
};

const isSingleFile = () =>
  fileStore.selectedCount === 1 &&
  !req.value?.items[fileStore.selected[0]].isDir;

const download = () => {
  if (!req.value) return false;

  if (isSingleFile()) {
    api.download(
      null,
      hash.value,
      token.value,
      req.value.items[fileStore.selected[0]].path
    );
    return true;
  }

  layoutStore.showHover({
    prompt: "download",
    confirm: (format: DownloadFormat) => {
      if (req.value === null) return false;
      layoutStore.closeHovers();

      let files: string[] = [];

      for (let i of fileStore.selected) {
        files.push(req.value.items[i].path);
      }

      api.download(format, hash.value, token.value, ...files);
      return true;
    },
  });

  return true;
};

const linkSelected = () => {
  return isSingleFile() && req.value
    ? api.getDownloadURL({
        ...req.value,
        hash: hash.value,
        path: req.value.items[fileStore.selected[0]].path,
      })
    : "";
};

const copyToClipboard = (text: string) => {
  copy(text).then(
    () => {
      // clipboard successfully set
      $showSuccess(t("success.linkCopied"));
    },
    () => {
      // clipboard write failed
    }
  );
};

onMounted(async () => {
  // Created
  hash.value = route.params.path[0];
  window.addEventListener("keydown", keyEvent);
  await fetchData();
  fileStore.selected[0] = 0;
});

onBeforeUnmount(() => {
  // Destroyed
  window.removeEventListener("keydown", keyEvent);
});
</script>
