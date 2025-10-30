<template>
  <div class="app-shell">
    <aside class="sidebar">
      <section class="panel-config">
        <header>
          <h1>Panel 配置</h1>
          <p>定义画布尺寸与布局，拖拽右侧面板中的图标来自由排布。</p>
        </header>
        <label class="field">
          <span>选择 Panel</span>
          <select v-model.number="panelIndex">
            <option v-for="(panel, idx) in panels" :key="panel.index" :value="idx">
              {{ labelForPanel(panel) }}
            </option>
          </select>
        </label>
        <div class="field-grid">
          <label class="field">
            <span>宽度 (px)</span>
            <input type="number" min="320" :value="panelWidth" @input="panelWidth = parseFloat($event.target.value)" />
          </label>
          <label class="field">
            <span>高度 (px)</span>
            <input type="number" min="320" :value="panelHeight" @input="panelHeight = parseFloat($event.target.value)" />
          </label>
        </div>
        <button class="ghost" @click="addPanel">新增 Panel</button>
      </section>

      <section class="item-config">
        <header>
          <h2>新建元素</h2>
          <p>设置类型与尺寸，加入后可在画布内拖拽与缩放。</p>
        </header>
        <div class="field-grid">
          <label class="field">
            <span>类型</span>
            <select v-model.number="newItem.type">
              <option v-for="option in typeOptions" :key="option.value" :value="option.value">
                {{ option.label }}
              </option>
            </select>
          </label>
          <label class="field">
            <span>宽度 (格)</span>
            <input type="number" min="1" v-model.number="newItem.widthUnits" />
          </label>
          <label class="field">
            <span>高度 (格)</span>
            <input type="number" min="1" v-model.number="newItem.heightUnits" />
          </label>
        </div>
        <label class="field">
          <span>名称</span>
          <input type="text" v-model="newItem.name" placeholder="应用名称" />
        </label>
        <label class="field">
          <span>包名</span>
          <input type="text" v-model="newItem.packageName" placeholder="com.example.app" />
        </label>
        <label class="field">
          <span>图标地址</span>
          <input type="url" v-model="newItem.iconUrl" placeholder="https://" />
        </label>
        <label class="field">
          <span>或上传图标</span>
          <input type="file" accept="image/*" @change="onIconUpload($event, newItem)" />
        </label>
        <button class="primary" @click="addItem">添加到当前 Panel</button>
      </section>

      <section class="item-detail" v-if="selectedItem">
        <header>
          <h2>选中元素</h2>
          <p>编辑基础信息或删除元素。</p>
        </header>
        <div class="field-grid">
          <label class="field">
            <span>类型</span>
            <select :value="selectedItem.type" @change="updateSelected('type', Number($event.target.value))">
              <option v-for="option in typeOptions" :key="option.value" :value="option.value">
                {{ option.label }}
              </option>
            </select>
          </label>
          <label class="field">
            <span>宽度 (格)</span>
            <input type="number" min="1" :value="selectedSize.width" @input="updateSelectedSize('width', $event.target.value)" />
          </label>
          <label class="field">
            <span>高度 (格)</span>
            <input type="number" min="1" :value="selectedSize.height" @input="updateSelectedSize('height', $event.target.value)" />
          </label>
        </div>
        <label class="field">
          <span>名称</span>
          <input type="text" :value="selectedItem.name" @input="updateSelected('name', $event.target.value)" />
        </label>
        <label class="field">
          <span>包名</span>
          <input type="text" :value="selectedItem.packageName" @input="updateSelected('packageName', $event.target.value)" />
        </label>
        <label class="field">
          <span>图标地址</span>
          <input type="url" :value="selectedItem.iconUrl" @input="updateSelected('iconUrl', $event.target.value)" />
        </label>
        <label class="field">
          <span>或上传新图标</span>
          <input type="file" accept="image/*" @change="onIconUpload($event, selectedItem)" />
        </label>
        <button class="danger" @click="removeSelected">删除元素</button>
      </section>

      <section class="export-section">
        <header>
          <h2>导出 / API</h2>
          <p>下载配置或通过接口提交。</p>
        </header>
        <div class="export-actions">
          <button class="secondary" @click="exportJson">导出 JSON</button>
          <button class="ghost" @click="sendToApi">发送至 API</button>
        </div>
        <textarea readonly :value="exportPreview" rows="10"></textarea>
      </section>
    </aside>

    <main class="canvas-area">
      <PanelEditor
        v-if="currentPanel"
        :panel="currentPanel"
        :selected-item-id="selectedItemId"
        @select-item="onSelectItem"
        @update-items="applyGridUpdates"
      />
      <div v-else class="empty">请先创建 Panel。</div>
    </main>
  </div>
</template>

<script setup>
import { computed, reactive, ref, watch } from 'vue';
import PanelEditor from './components/PanelEditor.vue';
import { GRID_CELL_SIZE } from './constants.js';

let itemIdCounter = 0;

const panels = ref([
  {
    index: 0,
    width: 960,
    height: 640,
    items: [
      {
        id: String(++itemIdCounter),
        type: 0,
        x: 0,
        y: 0,
        width: GRID_CELL_SIZE * 2,
        height: GRID_CELL_SIZE * 2,
        name: '腾讯视频',
        iconUrl: 'https://qzonestyle.gtimg.cn/aoi/sola/20211222171552.png',
        packageName: 'com.tencent.video'
      }
    ]
  }
]);

const panelIndex = ref(0);
const selectedItemId = ref(null);

const currentPanel = computed(() => panels.value[panelIndex.value] ?? null);

watch(panelIndex, () => {
  selectedItemId.value = null;
});

const panelWidth = computed({
  get: () => currentPanel.value?.width ?? 800,
  set: (value) => {
    if (!currentPanel.value || Number.isNaN(value)) return;
    currentPanel.value.width = Math.max(value, GRID_CELL_SIZE);
  }
});

const panelHeight = computed({
  get: () => currentPanel.value?.height ?? 600,
  set: (value) => {
    if (!currentPanel.value || Number.isNaN(value)) return;
    currentPanel.value.height = Math.max(value, GRID_CELL_SIZE);
  }
});

const typeOptions = [
  { value: 0, label: '0 - 通用元素' },
  { value: 100, label: '100 - 系统应用栏' },
  { value: 101, label: '101 - 天气' },
  { value: 102, label: '102 - 释放内存' }
];

const newItem = reactive({
  type: 0,
  name: '',
  packageName: '',
  iconUrl: '',
  widthUnits: 2,
  heightUnits: 2
});

const labelForPanel = (panel) => {
  if (panel.index === 0) return 'Panel 0 (左)';
  if (panel.index === 1) return 'Panel 1 (中)';
  if (panel.index === 2) return 'Panel 2 (右)';
  return `Panel ${panel.index}`;
};

const addPanel = () => {
  const nextIndex = panels.value.length;
  panels.value.push({
    index: nextIndex,
    width: 960,
    height: 640,
    items: []
  });
  panelIndex.value = panels.value.length - 1;
};

const addItem = () => {
  if (!currentPanel.value) return;
  const width = Math.max(newItem.widthUnits, 1) * GRID_CELL_SIZE;
  const height = Math.max(newItem.heightUnits, 1) * GRID_CELL_SIZE;
  const panel = currentPanel.value;
  panel.items.push({
    id: String(++itemIdCounter),
    type: newItem.type,
    x: 0,
    y: 0,
    width,
    height,
    name: newItem.name || '新建元素',
    iconUrl: newItem.iconUrl,
    packageName: newItem.packageName,
    autoPosition: true
  });
  selectedItemId.value = panel.items[panel.items.length - 1].id;
};

const onIconUpload = async (event, target) => {
  const [file] = event.target.files || [];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = () => {
    const url = reader.result;
    if (target === newItem) {
      newItem.iconUrl = url;
    } else if (target && 'id' in target) {
      target.iconUrl = url;
    }
  };
  reader.readAsDataURL(file);
};

const selectedItem = computed(() => {
  if (!currentPanel.value) return null;
  return currentPanel.value.items.find((item) => item.id === selectedItemId.value) ?? null;
});

const selectedSize = computed(() => {
  if (!selectedItem.value) return { width: 1, height: 1 };
  return {
    width: Math.max(Math.round(selectedItem.value.width / GRID_CELL_SIZE), 1),
    height: Math.max(Math.round(selectedItem.value.height / GRID_CELL_SIZE), 1)
  };
});

const updateSelected = (field, value) => {
  if (!selectedItem.value) return;
  selectedItem.value[field] = value;
};

const updateSelectedSize = (field, value) => {
  if (!selectedItem.value) return;
  const numeric = Math.max(parseInt(value, 10) || 1, 1);
  selectedItem.value[field] = numeric * GRID_CELL_SIZE;
};

const removeSelected = () => {
  if (!currentPanel.value || !selectedItem.value) return;
  currentPanel.value.items = currentPanel.value.items.filter((item) => item.id !== selectedItemId.value);
  selectedItemId.value = null;
};

const onSelectItem = (id) => {
  selectedItemId.value = id;
};

const applyGridUpdates = (updates) => {
  if (!currentPanel.value) return;
  updates.forEach((update) => {
    const item = currentPanel.value.items.find((candidate) => candidate.id === update.id);
    if (item) {
      item.x = update.x;
      item.y = update.y;
      item.width = update.width;
      item.height = update.height;
    }
  });
};

const exportPreview = computed(() => {
  const sanitized = panels.value.map((panel) => ({
    index: panel.index,
    width: panel.width,
    height: panel.height,
    items: panel.items.map(({ id, autoPosition, ...rest }) => rest)
  }));
  return JSON.stringify(sanitized, null, 2);
});

const exportJson = () => {
  const blob = new Blob([exportPreview.value], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const anchor = document.createElement('a');
  anchor.href = url;
  anchor.download = 'panels.json';
  anchor.click();
  URL.revokeObjectURL(url);
};

const sendToApi = async () => {
  console.info('TODO: send payload to API endpoint', exportPreview.value);
  alert('已在控制台输出 JSON，可替换为真实接口调用。');
};
</script>

<style scoped>
.app-shell {
  display: grid;
  grid-template-columns: 360px 1fr;
  height: 100vh;
  overflow: hidden;
}

.sidebar {
  padding: 24px;
  overflow-y: auto;
  background: linear-gradient(180deg, rgba(79, 122, 254, 0.12), transparent 40%);
  border-right: 1px solid rgba(79, 122, 254, 0.1);
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.canvas-area {
  padding: 24px;
  background: #f4f6fb;
}

section {
  background: #fff;
  border-radius: 16px;
  padding: 20px;
  box-shadow: 0 10px 40px rgba(31, 42, 68, 0.08);
  display: flex;
  flex-direction: column;
  gap: 16px;
}

section header h1,
section header h2 {
  margin: 0;
  font-size: 18px;
  color: #1f2a44;
}

section header p {
  margin: 4px 0 0;
  font-size: 13px;
  color: #5f6b87;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 13px;
  color: #4f5d7a;
}

.field input,
.field select,
textarea {
  border: 1px solid rgba(79, 122, 254, 0.2);
  border-radius: 10px;
  padding: 8px 10px;
  font-size: 14px;
  background: #fff;
  color: #1f2a44;
}

textarea {
  font-family: 'Fira Code', 'Courier New', monospace;
  min-height: 160px;
  resize: vertical;
}

.field-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(120px, 1fr));
  gap: 12px;
}

button {
  border: none;
  border-radius: 10px;
  padding: 10px 16px;
  font-size: 14px;
  font-weight: 600;
  transition: all 0.2s ease;
}

button.primary {
  background: #4f7afe;
  color: #fff;
}

button.primary:hover {
  background: #3b63d8;
}

button.secondary {
  background: rgba(79, 122, 254, 0.12);
  color: #1f2a44;
}

button.ghost {
  background: transparent;
  color: #4f7afe;
  border: 1px dashed rgba(79, 122, 254, 0.4);
}

button.danger {
  background: rgba(244, 63, 94, 0.1);
  color: #d61f45;
}

button:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 16px rgba(31, 42, 68, 0.1);
}

.export-section textarea {
  width: 100%;
}

.empty {
  display: grid;
  place-items: center;
  color: #5f6b87;
  font-size: 18px;
  height: 100%;
}
</style>
