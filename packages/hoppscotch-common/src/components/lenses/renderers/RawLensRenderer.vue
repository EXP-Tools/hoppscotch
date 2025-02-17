<template>
  <div class="flex flex-1 flex-col">
    <div
      class="sticky top-lowerSecondaryStickyFold z-10 flex flex-shrink-0 items-center justify-between overflow-x-auto border-b border-dividerLight bg-primary pl-4"
    >
      <label class="truncate font-semibold text-secondaryLight">
        {{ t("response.body") }}
      </label>
      <div class="flex">
        <HoppButtonSecondary
          v-if="response.body"
          v-tippy="{ theme: 'tooltip' }"
          :title="t('state.linewrap')"
          :class="{ '!text-accent': WRAP_LINES }"
          :icon="IconWrapText"
          @click.prevent="toggleNestedSetting('WRAP_LINES', 'httpResponseBody')"
        />
        <HoppButtonSecondary
          v-if="response.body"
          v-tippy="{ theme: 'tooltip', allowHTML: true }"
          :title="`${t(
            'action.download_file'
          )} <kbd>${getSpecialKey()}</kbd><kbd>J</kbd>`"
          :icon="downloadIcon"
          @click="downloadResponse"
        />
        <HoppButtonSecondary
          v-if="response.body"
          v-tippy="{ theme: 'tooltip', allowHTML: true }"
          :title="`${t(
            'action.copy'
          )} <kbd>${getSpecialKey()}</kbd><kbd>.</kbd>`"
          :icon="copyIcon"
          @click="copyResponse"
        />
      </div>
    </div>
    <div ref="rawResponse" class="flex flex-1 flex-col"></div>
  </div>
</template>

<script setup lang="ts">
import IconWrapText from "~icons/lucide/wrap-text"
import { ref, computed, reactive } from "vue"
import { flow, pipe } from "fp-ts/function"
import * as S from "fp-ts/string"
import * as RNEA from "fp-ts/ReadonlyNonEmptyArray"
import * as A from "fp-ts/Array"
import * as O from "fp-ts/Option"
import { useCodemirror } from "@composables/codemirror"
import { HoppRESTResponse } from "~/helpers/types/HoppRESTResponse"
import { useI18n } from "@composables/i18n"
import {
  useResponseBody,
  useDownloadResponse,
  useCopyResponse,
} from "@composables/lens-actions"
import { objFieldMatches } from "~/helpers/functional/object"
import { defineActionHandler } from "~/helpers/actions"
import { getPlatformSpecialKey as getSpecialKey } from "~/helpers/platformutils"
import { useNestedSetting } from "~/composables/settings"
import { toggleNestedSetting } from "~/newstore/settings"

const t = useI18n()

const props = defineProps<{
  response: HoppRESTResponse & { type: "success" | "fail" }
}>()

const { responseBodyText } = useResponseBody(props.response)

const rawResponseBody = computed(() =>
  props.response.type === "fail" || props.response.type === "success"
    ? props.response.body
    : new ArrayBuffer(0)
)

const responseType = computed(() =>
  pipe(
    props.response,
    O.fromPredicate(objFieldMatches("type", ["fail", "success"] as const)),
    O.chain(
      // Try getting content-type
      flow(
        (res) => res.headers,
        A.findFirst((h) => h.key.toLowerCase() === "content-type"),
        O.map(flow((h) => h.value, S.split(";"), RNEA.head, S.toLowerCase))
      )
    ),
    O.getOrElse(() => "text/plain")
  )
)

const { downloadIcon, downloadResponse } = useDownloadResponse(
  responseType.value,
  rawResponseBody
)

const { copyIcon, copyResponse } = useCopyResponse(responseBodyText)

const rawResponse = ref<any | null>(null)
const WRAP_LINES = useNestedSetting("WRAP_LINES", "httpResponseBody")

useCodemirror(
  rawResponse,
  responseBodyText,
  reactive({
    extendedEditorConfig: {
      mode: "text/plain",
      readOnly: true,
      lineWrapping: WRAP_LINES,
    },
    linter: null,
    completer: null,
    environmentHighlights: true,
  })
)

defineActionHandler("response.file.download", () => downloadResponse())
defineActionHandler("response.copy", () => copyResponse())
</script>

<style lang="scss" scoped>
:deep(.cm-panels) {
  @apply top-lowerTertiaryStickyFold #{!important};
}
</style>
