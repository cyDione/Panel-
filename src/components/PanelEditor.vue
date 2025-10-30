<template>
  <div class="panel-editor">
    <Zoomable
      class="zoom-wrapper"
      :zoom="zoom"
      :min-zoom="0.4"
      :max-zoom="2"
      @zoom="onZoom"
    >
      <div
        class="panel-canvas"
        :style="{
          width: panel.width + 'px',
          height: panel.height + 'px'
        }"
      >
        <div ref="gridRef" class="grid-stack"></div>
      </div>
    </Zoomable>
    <div class="zoom-controls">
      <label>缩放: {{ Math.round(zoom * 100) }}%</label>
      <input type="range" min="0.4" max="2" step="0.1" v-model.number="zoom" />
    </div>
  </div>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue';
import Zoomable from 'vue-zoomable';
import { GridStack } from 'gridstack';
import 'gridstack/dist/h5/gridstack-dd-native';
import { GRID_CELL_SIZE } from '../constants.js';

const props = defineProps({
  panel: {
    type: Object,
    required: true
  },
  selectedItemId: {
    type: [String, Number, null],
    default: null
  }
});

const emit = defineEmits(['select-item', 'update-items']);

const zoom = ref(1);
const gridRef = ref(null);
let gridInstance;
let isRendering = false;

const columns = computed(() => Math.max(Math.round(props.panel.width / GRID_CELL_SIZE), 1));

const toGridUnits = (value) => Math.max(Math.round(value / GRID_CELL_SIZE), 1);
const toGridPosition = (value) => Math.max(Math.round(value / GRID_CELL_SIZE), 0);
const toPixels = (value) => value * GRID_CELL_SIZE;

const ensureItemDefaults = (item) => {
  const normalized = { ...item };
  normalized.width = normalized.width || GRID_CELL_SIZE * 2;
  normalized.height = normalized.height || GRID_CELL_SIZE * 2;
  normalized.x = normalized.x || 0;
  normalized.y = normalized.y || 0;
  return normalized;
};

const renderGrid = () => {
  if (!gridInstance) return;
  isRendering = true;
  gridInstance.removeAll(false);

  try {
    props.panel.items.forEach((rawItem) => {
    const item = ensureItemDefaults(rawItem);
    const wrapper = document.createElement('div');
    wrapper.className = 'grid-stack-item';
    wrapper.setAttribute('gs-x', String(toGridPosition(item.x)));
    wrapper.setAttribute('gs-y', String(toGridPosition(item.y)));
    wrapper.setAttribute('gs-width', String(toGridUnits(item.width)));
    wrapper.setAttribute('gs-height', String(toGridUnits(item.height)));
    wrapper.setAttribute('data-item-id', String(item.id));
    if (item.autoPosition) {
      wrapper.setAttribute('gs-auto-position', 'true');
    }

    const content = document.createElement('div');
    content.className = 'grid-stack-item-content';
    if (props.selectedItemId === item.id) {
      content.classList.add('is-selected');
    }

    const header = document.createElement('div');
    header.className = 'item-header';

    const icon = document.createElement('img');
    icon.alt = item.name || 'icon';
    icon.src = item.iconUrl || 'https://via.placeholder.com/80?text=Icon';
    header.appendChild(icon);

    const titleWrap = document.createElement('div');
    titleWrap.className = 'item-title';
    const title = document.createElement('div');
    title.className = 'item-name';
    title.textContent = item.name || '未命名应用';
    titleWrap.appendChild(title);

    const subtitle = document.createElement('div');
    subtitle.className = 'item-type';
    subtitle.textContent = `类型: ${item.type}`;
    titleWrap.appendChild(subtitle);

    header.appendChild(titleWrap);

    const meta = document.createElement('div');
    meta.className = 'item-meta';
    const packageRow = document.createElement('div');
    packageRow.textContent = `包名: ${item.packageName || '-'}`;
    meta.appendChild(packageRow);

    const sizeRow = document.createElement('div');
    sizeRow.textContent = `尺寸: ${Math.round(item.width)} x ${Math.round(item.height)}`;
    meta.appendChild(sizeRow);

    content.appendChild(header);
    content.appendChild(meta);

    content.addEventListener('click', (event) => {
      event.stopPropagation();
      emit('select-item', item.id);
    });

    wrapper.appendChild(content);
      const node = gridInstance.addWidget(wrapper);
      if (item.autoPosition && node) {
        rawItem.autoPosition = false;
        rawItem.x = toPixels(node.x ?? 0);
        rawItem.y = toPixels(node.y ?? 0);
        rawItem.width = toPixels(node.w ?? toGridUnits(item.width));
        rawItem.height = toPixels(node.h ?? toGridUnits(item.height));
      }
    });
  } finally {
    isRendering = false;
  }
};

const onGridChange = (_event, changedNodes) => {
  if (isRendering) return;
  if (!changedNodes || changedNodes.length === 0) return;
  const updates = changedNodes
    .map((node) => {
      const id = node.el?.getAttribute('data-item-id');
      if (!id) return null;
      return {
        id,
        x: toPixels(node.x),
        y: toPixels(node.y),
        width: toPixels(node.w),
        height: toPixels(node.h)
      };
    })
    .filter(Boolean);
  if (updates.length) {
    emit('update-items', updates);
  }
};

const initializeGrid = () => {
  if (!gridRef.value) return;
  gridInstance = GridStack.init(
    {
      float: false,
      margin: 8,
      cellHeight: GRID_CELL_SIZE,
      column: columns.value,
      minRow: Math.max(Math.round(props.panel.height / GRID_CELL_SIZE), 1),
      resizable: { handles: 'all' },
      draggable: { scroll: true }
    },
    gridRef.value
  );

  gridInstance.on('change', onGridChange);
  renderGrid();
};

const updateGridDimensions = () => {
  if (!gridInstance) return;
  gridInstance.cellHeight(GRID_CELL_SIZE);
  gridInstance.column(columns.value);
  gridInstance.opts.minRow = Math.max(Math.round(props.panel.height / GRID_CELL_SIZE), 1);
  renderGrid();
};

onMounted(() => {
  initializeGrid();
});

onBeforeUnmount(() => {
  gridInstance?.destroy(false);
});

watch(
  () => props.panel.items,
  () => {
    if (!isRendering) {
      renderGrid();
    }
  },
  { deep: true }
);

watch(
  () => [props.panel.width, props.panel.height],
  () => {
    updateGridDimensions();
  }
);

watch(
  () => props.selectedItemId,
  () => {
    renderGrid();
  }
);

const onZoom = (value) => {
  zoom.value = value;
};
</script>

<style scoped>
.panel-editor {
  display: flex;
  flex-direction: column;
  gap: 12px;
  height: 100%;
}

.zoom-wrapper {
  flex: 1;
  overflow: hidden;
  border-radius: 16px;
  background: #fff;
  box-shadow: inset 0 0 0 1px rgba(79, 122, 254, 0.08);
}

.panel-canvas {
  border: 2px dashed rgba(79, 122, 254, 0.4);
  border-radius: 16px;
  background: repeating-linear-gradient(
      0deg,
      rgba(79, 122, 254, 0.05),
      rgba(79, 122, 254, 0.05) 1px,
      transparent 1px,
      transparent 80px
    ),
    repeating-linear-gradient(
      90deg,
      rgba(79, 122, 254, 0.05),
      rgba(79, 122, 254, 0.05) 1px,
      transparent 1px,
      transparent 80px
    );
  position: relative;
  margin: 0 auto;
}

.zoom-controls {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 14px;
  color: #4f5d7a;
}

.zoom-controls input[type='range'] {
  flex: 1;
}

.grid-stack-item-content.is-selected {
  outline: 3px solid #4f7afe;
}
</style>
