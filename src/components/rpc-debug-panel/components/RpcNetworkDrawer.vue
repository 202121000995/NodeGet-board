<script setup lang="ts">
import { computed, ref, watch } from "vue";
import { useDraggable, useEventListener, useWindowSize } from "@vueuse/core";
import { Copy, FileJson, Pencil, X } from "lucide-vue-next";
import type { RpcDebugRecord } from "../rpcDebugStore";
import { formatDebugPayload } from "../rpcDebugStore";
import { statusText } from "../helpers";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import {
  Tooltip,
  TooltipContent,
  TooltipProvider,
  TooltipTrigger,
} from "@/components/ui/tooltip";
import { cn } from "@/lib/utils";

type DetailTabKey = "headers" | "payload" | "response";

const detailTabs: Array<{ key: DetailTabKey; label: string }> = [
  { key: "headers", label: "标头" },
  { key: "payload", label: "负载" },
  { key: "response", label: "响应" },
];

const DEFAULT_DRAWER_WIDTH = 420;
const MIN_DRAWER_WIDTH = 360;
const MAX_DRAWER_WIDTH = 880;
const ACTION_TEXT_DRAWER_WIDTH = 560;

const props = withDefaults(
  defineProps<{
    record: RpcDebugRecord;
    class?: string;
    width?: number;
  }>(),
  {
    width: DEFAULT_DRAWER_WIDTH,
  },
);

const emit = defineEmits<{
  close: [];
  copy: [text: string, message?: string];
  edit: [record: RpcDebugRecord];
  "update:width": [width: number];
}>();

const activeDetailTab = ref<DetailTabKey>("headers");
const drawerResizeHandle = ref<HTMLElement | null>(null);

let resizeStartX = 0;
let resizeStartWidth = DEFAULT_DRAWER_WIDTH;

const { width: windowWidth } = useWindowSize();

const requestText = computed(() =>
  formatDebugPayload(props.record.request, props.record.method),
);

const responseText = computed(() =>
  formatDebugPayload(props.record.response, props.record.method),
);

const drawerWidth = computed({
  get: () => props.width,
  set: (width) => emit("update:width", clampDrawerWidth(width)),
});

const drawerStyle = computed(() => ({
  width: `${drawerWidth.value}px`,
}));

const showActionText = computed(
  () => drawerWidth.value >= ACTION_TEXT_DRAWER_WIDTH,
);

const statusBadgeVariant = computed(() => {
  if (
    props.record.status === "success" ||
    props.record.status === "streaming"
  ) {
    return "default";
  }
  if (props.record.status === "error" || props.record.status === "closed") {
    return "destructive";
  }
  if (props.record.status === "pending") {
    return "secondary";
  }
  return "outline";
});

const headerRows = computed(() => [
  { label: "方法", value: props.record.method },
  { label: "状态", value: statusText(props.record) },
  { label: "类型", value: props.record.kind },
  { label: "方向", value: props.record.direction },
  { label: "请求 ID", value: props.record.id ?? "-" },
  { label: "订阅 ID", value: props.record.subscription ?? "-" },
  { label: "连接 ID", value: `#${props.record.connectionId}` },
  { label: "URL", value: props.record.url },
  { label: "开始时间", value: formatTimestamp(props.record.startedAt) },
  { label: "结束时间", value: formatTimestamp(props.record.endedAt) },
  {
    label: "耗时",
    value:
      props.record.durationMs != null ? `${props.record.durationMs}ms` : "-",
  },
  { label: "备注", value: props.record.note ?? "-" },
]);

function formatTimestamp(timestamp?: number) {
  if (!timestamp) return "-";
  return new Intl.DateTimeFormat("zh-CN", {
    hour: "2-digit",
    minute: "2-digit",
    second: "2-digit",
  }).format(timestamp);
}

function headerText() {
  return headerRows.value
    .map((row) => `${row.label}: ${String(row.value)}`)
    .join("\n");
}

function activeTabCopyText() {
  if (activeDetailTab.value === "payload") return requestText.value;
  if (activeDetailTab.value === "response") return responseText.value;
  return headerText();
}

function activeTabCopyMessage() {
  if (activeDetailTab.value === "payload") return "负载已复制";
  if (activeDetailTab.value === "response") return "响应已复制";
  return "标头已复制";
}

function copyActiveTab() {
  emit("copy", activeTabCopyText(), activeTabCopyMessage());
}

function maxDrawerWidth() {
  return Math.min(
    MAX_DRAWER_WIDTH,
    Math.max(MIN_DRAWER_WIDTH, windowWidth.value - 48),
  );
}

function clampDrawerWidth(width: number) {
  return Math.min(Math.max(width, MIN_DRAWER_WIDTH), maxDrawerWidth());
}

function handleDrawerEscapeKeydown(event: KeyboardEvent) {
  if (event.key !== "Escape") return;
  event.preventDefault();
  event.stopPropagation();
  event.stopImmediatePropagation();
  emit("close");
}

const { isDragging: isResizingDrawer } = useDraggable(drawerResizeHandle, {
  axis: "x",
  preventDefault: true,
  stopPropagation: true,
  onStart(_position, event) {
    resizeStartX = event.clientX;
    resizeStartWidth = drawerWidth.value;
  },
  onMove(_position, event) {
    drawerWidth.value = clampDrawerWidth(
      resizeStartWidth + resizeStartX - event.clientX,
    );
  },
});

watch(windowWidth, () => {
  drawerWidth.value = clampDrawerWidth(drawerWidth.value);
});

useEventListener("keydown", handleDrawerEscapeKeydown, { capture: true });
</script>

<template>
  <aside
    :class="
      cn(
        'absolute top-0 right-0 bottom-0 flex max-w-full flex-col border-l bg-background shadow-xl',
        props.class,
      )
    "
    :style="drawerStyle"
  >
    <div
      ref="drawerResizeHandle"
      title="拖拽调整详情面板宽度"
      class="group absolute top-0 left-0 z-20 h-full w-2 -translate-x-1 cursor-ew-resize touch-none outline-none"
    >
      <div
        :class="
          cn(
            'mx-auto h-full w-px bg-border transition-colors group-hover:bg-primary/60 group-focus-visible:bg-primary',
            isResizingDrawer && 'bg-primary',
          )
        "
      />
    </div>

    <div class="border-b px-4 py-3">
      <div class="flex items-start justify-between gap-3">
        <div class="min-w-0 space-y-1">
          <div class="flex min-w-0 flex-wrap items-center gap-1.5">
            <h2 class="shrink-0 text-sm font-semibold">消息详情</h2>
            <Badge
              :variant="statusBadgeVariant"
              class="h-5 rounded px-1.5 text-[10px]"
            >
              {{ statusText(record) }}
            </Badge>
            <Badge variant="outline" class="h-5 rounded px-1.5 text-[10px]">
              {{ record.durationMs != null ? `${record.durationMs}ms` : "-" }}
            </Badge>
            <Badge variant="outline" class="h-5 rounded px-1.5 text-[10px]">
              {{ record.direction }}
            </Badge>
          </div>
          <p class="max-w-full truncate text-xs text-muted-foreground">
            {{ record.method }}
          </p>
        </div>
        <Button
          variant="ghost"
          size="icon-sm"
          class="shrink-0"
          @click="emit('close')"
        >
          <X class="size-4" />
        </Button>
      </div>
    </div>

    <Tabs v-model="activeDetailTab" class="flex min-h-0 flex-1 flex-col">
      <div
        class="flex h-10 shrink-0 items-center justify-between border-b px-4"
      >
        <TabsList class="h-8 rounded-md p-0.5">
          <TabsTrigger
            v-for="tab in detailTabs"
            :key="tab.key"
            :value="tab.key"
            class="h-7 px-2.5 text-xs"
          >
            {{ tab.label }}
          </TabsTrigger>
        </TabsList>
        <div v-if="showActionText" class="flex items-center gap-1.5">
          <Button
            aria-label="完整复制"
            variant="ghost"
            size="sm"
            class="h-8 gap-1.5 px-2.5 text-xs"
            @click="
              emit(
                'copy',
                formatDebugPayload(record, record.method),
                '完整记录已复制',
              )
            "
          >
            <FileJson class="size-3.5" />
            完整复制
          </Button>
          <Button
            aria-label="编辑"
            variant="ghost"
            size="sm"
            class="h-8 gap-1.5 px-2.5 text-xs"
            @click="emit('edit', record)"
          >
            <Pencil class="size-3.5" />
            编辑
          </Button>
          <Button
            aria-label="复制当前标签内容"
            variant="ghost"
            size="sm"
            class="h-8 gap-1.5 px-2.5 text-xs"
            @click="copyActiveTab"
          >
            <Copy class="size-3.5" />
            复制当前标签
          </Button>
        </div>
        <TooltipProvider v-else :delay-duration="120">
          <div class="flex items-center gap-1.5">
            <Tooltip>
              <TooltipTrigger as-child>
                <Button
                  aria-label="完整复制"
                  variant="ghost"
                  size="icon-sm"
                  @click="
                    emit(
                      'copy',
                      formatDebugPayload(record, record.method),
                      '完整记录已复制',
                    )
                  "
                >
                  <FileJson class="size-3.5" />
                </Button>
              </TooltipTrigger>
              <TooltipContent side="bottom">完整复制</TooltipContent>
            </Tooltip>
            <Tooltip>
              <TooltipTrigger as-child>
                <Button
                  aria-label="编辑"
                  variant="ghost"
                  size="icon-sm"
                  @click="emit('edit', record)"
                >
                  <Pencil class="size-3.5" />
                </Button>
              </TooltipTrigger>
              <TooltipContent side="bottom">编辑</TooltipContent>
            </Tooltip>
            <Tooltip>
              <TooltipTrigger as-child>
                <Button
                  aria-label="复制当前标签内容"
                  variant="ghost"
                  size="icon-sm"
                  @click="copyActiveTab"
                >
                  <Copy class="size-3.5" />
                </Button>
              </TooltipTrigger>
              <TooltipContent side="bottom">复制当前标签内容</TooltipContent>
            </Tooltip>
          </div>
        </TooltipProvider>
      </div>

      <div class="min-h-0 flex-1 overflow-auto p-4">
        <TabsContent
          value="headers"
          class="m-0 grid grid-cols-[88px_minmax(0,1fr)] gap-x-4 gap-y-2 text-sm"
        >
          <template v-for="row in headerRows" :key="row.label">
            <dt class="text-muted-foreground">{{ row.label }}</dt>
            <dd class="min-w-0 font-mono text-xs leading-6 break-all">
              {{ row.value }}
            </dd>
          </template>
        </TabsContent>

        <TabsContent value="payload" class="m-0 h-full min-h-0">
          <div class="overflow-hidden rounded-md border bg-muted/35">
            <pre class="min-h-full overflow-auto p-3 text-xs leading-relaxed">{{
              requestText || "无发送内容"
            }}</pre>
          </div>
        </TabsContent>

        <TabsContent value="response" class="m-0 h-full min-h-0">
          <div class="overflow-hidden rounded-md border bg-muted/35">
            <pre class="min-h-full overflow-auto p-3 text-xs leading-relaxed">{{
              responseText || "等待响应"
            }}</pre>
          </div>
        </TabsContent>
      </div>
    </Tabs>
  </aside>
</template>
