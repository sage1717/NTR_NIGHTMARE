<template>
  <div class="creator-root">
    <!-- 已提交状态：显示角色摘要 -->
    <div v-if="submitted" class="summary-view">
      <div class="summary-header">
        <span class="glitch-icon">◈</span>
        <h2>角色已创建</h2>
        <span class="glitch-icon">◈</span>
      </div>
      <div class="summary-grid">
        <div v-for="(value, key) in summaryFields" :key="key" class="summary-item">
          <span class="summary-label">{{ key }}</span>
          <span class="summary-value">{{ value }}</span>
        </div>
      </div>
      <p class="summary-hint">冒险已开始，请继续对话…</p>
    </div>

    <!-- 表单状态 -->
    <div v-else class="form-view">
      <div class="form-header">
        <div class="header-deco">▸ ▸ ▸</div>
        <h1 class="title">创建角色</h1>
        <p class="subtitle">填写你的角色信息，留空部分将由 AI 随机补全</p>
        <div class="header-deco">◂ ◂ ◂</div>
      </div>

      <div class="form-body">
        <!-- 第一行：姓名 + 年龄 -->
        <div class="form-row">
          <div class="field">
            <label class="field-label">姓名</label>
            <input
              v-model="form.name"
              class="field-input"
              type="text"
              placeholder="留空随机生成"
              :disabled="isSubmitting"
            />
          </div>
          <div class="field field--sm">
            <label class="field-label">年龄</label>
            <input
              v-model="form.age"
              class="field-input"
              type="text"
              placeholder="随机"
              :disabled="isSubmitting"
            />
          </div>
        </div>

        <!-- 第二行：性别 + 种族 -->
        <div class="form-row">
          <div class="field field--sm">
            <label class="field-label">性别</label>
            <div class="gender-group">
              <button
                v-for="g in genderOptions"
                :key="g.value"
                class="gender-btn"
                :class="{ 'gender-btn--active': form.gender === g.value }"
                :disabled="isSubmitting"
                @click="form.gender = g.value"
              >{{ g.label }}</button>
            </div>
          </div>
          <div class="field">
            <label class="field-label">种族</label>
            <input
              v-model="form.race"
              class="field-input"
              type="text"
              placeholder="人类、精灵、兽人… 留空随机"
              :disabled="isSubmitting"
            />
          </div>
        </div>

        <!-- 外貌 -->
        <div class="field field--full">
          <label class="field-label">外貌</label>
          <textarea
            v-model="form.appearance"
            class="field-textarea"
            rows="3"
            placeholder="描述角色的外貌特征，留空随机生成"
            :disabled="isSubmitting"
          />
        </div>

        <!-- 特质 -->
        <div class="field field--full">
          <label class="field-label">特质</label>
          <textarea
            v-model="form.traits"
            class="field-textarea"
            rows="3"
            placeholder="性格、能力、习惯等特点，留空随机生成"
            :disabled="isSubmitting"
          />
        </div>

        <!-- 背景故事 -->
        <div class="field field--full">
          <label class="field-label">背景故事 <span class="field-label--extra">（额外设定补充）</span></label>
          <textarea
            v-model="form.background"
            class="field-textarea"
            rows="4"
            placeholder="角色的来历、经历或额外设定，留空随机生成"
            :disabled="isSubmitting"
          />
        </div>
      </div>

      <!-- 提交按钮 -->
      <div class="form-footer">
        <button class="submit-btn" :disabled="isSubmitting" @click="handleSubmit">
          <span v-if="isSubmitting" class="btn-inner">
            <span class="spinner">⬡</span> 生成中…
          </span>
          <span v-else class="btn-inner">
            ▶ 开始冒险
          </span>
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
// ── 常量 ──────────────────────────────────────────────────────────────────
const CHAT_VAR_KEY = 'starter_character_created';

const genderOptions = [
  { label: '男', value: '男' },
  { label: '女', value: '女' },
  { label: '随机', value: '' },
] as const;

// ── 状态 ──────────────────────────────────────────────────────────────────
const form = ref({
  name: '',
  gender: '' as '' | '男' | '女',
  race: '',
  age: '',
  appearance: '',
  traits: '',
  background: '',
});

const isSubmitting = ref(false);
const submitted = ref(false);

// ── 初始化：检查是否已创建过 ────────────────────────────────────────────
onMounted(() => {
  const vars = getVariables({ type: 'chat' }) as Record<string, unknown>;
  if (_.get(vars, CHAT_VAR_KEY)) {
    const saved = _.get(vars, 'starter_character_form') as typeof form.value | undefined;
    if (saved) form.value = saved;
    submitted.value = true;
  }
});

// ── 摘要显示字段（仅非空） ───────────────────────────────────────────────
const summaryFields = computed(() => {
  const f = form.value;
  const map: Record<string, string> = {};
  if (f.name)       map['姓名']     = f.name;
  if (f.gender)     map['性别']     = f.gender;
  if (f.race)       map['种族']     = f.race;
  if (f.age)        map['年龄']     = f.age;
  if (f.appearance) map['外貌']     = f.appearance;
  if (f.traits)     map['特质']     = f.traits;
  if (f.background) map['背景故事'] = f.background;
  return map;
});

// ── 提交逻辑 ─────────────────────────────────────────────────────────────
async function handleSubmit() {
  isSubmitting.value = true;

  try {
    // 构建角色描述文本
    const lines: string[] = [];
    const f = form.value;

    lines.push('【角色创建】');
    lines.push(f.name       ? `姓名：${f.name}`           : '姓名：（请随机生成）');
    lines.push(f.gender     ? `性别：${f.gender}`         : '性别：（请随机生成）');
    lines.push(f.race       ? `种族：${f.race}`           : '种族：（请随机生成）');
    lines.push(f.age        ? `年龄：${f.age}`            : '年龄：（请随机生成）');
    lines.push(f.appearance ? `外貌：${f.appearance}`     : '外貌：（请随机生成）');
    lines.push(f.traits     ? `特质：${f.traits}`         : '特质：（请随机生成）');
    if (f.background) lines.push(`背景故事：${f.background}`);

    lines.push('');
    lines.push('请根据以上信息补全所有「请随机生成」的字段，并以此角色身份开始冒险。');

    const user_input = lines.join('\n');

    // 持久化状态（先存，防止 generate 中途跳转）
    replaceVariables(
      { [CHAT_VAR_KEY]: true, starter_character_form: form.value },
      { type: 'chat' },
    );

    // 请求 AI 生成开场
    await generate({ user_input });

    submitted.value = true;
  } catch (err) {
    console.error('[创建角色] 生成失败', err);
  } finally {
    isSubmitting.value = false;
  }
}
</script>

<style lang="scss" scoped>
// ── 根容器（CSS 变量定义在此，scoped 内有效） ─────────────────────────────
.creator-root {
  --green: #00ff41;
  --green-dim: #00cc33;
  --green-glow: rgba(0, 255, 65, 0.35);
  --bg: #000000;
  --bg-panel: #080808;
  --bg-field: #0a0a0a;
  --border: rgba(0, 255, 65, 0.4);
  --border-hover: rgba(0, 255, 65, 0.8);
  --font: 'Microsoft YaHei', '微软雅黑', sans-serif;
  --radius: 4px;

  width: 100%;
  background: var(--bg);
  color: var(--green);
  font-family: var(--font);
  font-size: 13px;
  line-height: 1.6;
  padding: 16px;
  box-sizing: border-box;
}

// ── 已提交摘要 ─────────────────────────────────────────────────────────────
.summary-view {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}

.summary-header {
  display: flex;
  align-items: center;
  gap: 10px;

  h2 {
    font-family: var(--font);
    font-weight: 700;
    font-size: 16px;
    color: var(--green);
    margin: 0;
    text-shadow: 0 0 8px var(--green-glow);
  }

  .glitch-icon {
    color: var(--green-dim);
    font-size: 14px;
  }
}

.summary-grid {
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 6px;
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 10px 12px;
  background: var(--bg-panel);
}

.summary-item {
  display: flex;
  gap: 12px;
  font-size: 12px;

  .summary-label {
    font-weight: 700;
    color: var(--green);
    min-width: 56px;
    flex-shrink: 0;
  }

  .summary-value {
    color: rgba(0, 255, 65, 0.75);
    word-break: break-word;
  }
}

.summary-hint {
  font-size: 11px;
  color: rgba(0, 255, 65, 0.45);
  margin: 0;
  letter-spacing: 0.04em;
}

// ── 表单视图 ──────────────────────────────────────────────────────────────
.form-view {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.form-header {
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;

  .header-deco {
    font-size: 10px;
    color: var(--green-dim);
    letter-spacing: 6px;
    opacity: 0.6;
  }
}

.title {
  font-family: var(--font);
  font-weight: 700;
  font-size: 20px;
  color: var(--green);
  margin: 4px 0;
  text-shadow: 0 0 12px var(--green-glow), 0 0 24px rgba(0, 255, 65, 0.15);
  letter-spacing: 0.12em;
}

.subtitle {
  font-size: 11px;
  color: rgba(0, 255, 65, 0.5);
  margin: 0;
  letter-spacing: 0.04em;
}

// ── 表单主体 ──────────────────────────────────────────────────────────────
.form-body {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.form-row {
  display: flex;
  gap: 10px;
}

// ── 字段 ──────────────────────────────────────────────────────────────────
.field {
  display: flex;
  flex-direction: column;
  gap: 4px;
  flex: 1;

  &--sm {
    flex: 0 0 110px;
  }

  &--full {
    width: 100%;
  }
}

.field-label {
  font-family: var(--font);
  font-weight: 700;
  font-size: 11px;
  color: var(--green);
  text-transform: uppercase;
  letter-spacing: 0.08em;

  &--extra {
    font-weight: 400;
    font-size: 10px;
    color: rgba(0, 255, 65, 0.45);
    text-transform: none;
    letter-spacing: 0;
  }
}

%input-base {
  width: 100%;
  background: var(--bg-field);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  color: #ccffcc;
  font-family: 'Consolas', 'Courier New', monospace;
  font-size: 12px;
  padding: 6px 8px;
  box-sizing: border-box;
  outline: none;
  transition: border-color 0.15s, box-shadow 0.15s;
  resize: none;

  &::placeholder {
    color: rgba(0, 255, 65, 0.2);
  }

  &:focus {
    border-color: var(--border-hover);
    box-shadow: 0 0 6px var(--green-glow);
  }

  &:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }
}

.field-input {
  @extend %input-base;
}

.field-textarea {
  @extend %input-base;
  line-height: 1.5;
}

// ── 性别选择 ──────────────────────────────────────────────────────────────
.gender-group {
  display: flex;
  gap: 5px;
}

.gender-btn {
  flex: 1;
  padding: 5px 4px;
  background: transparent;
  border: 1px solid var(--border);
  border-radius: var(--radius);
  color: rgba(0, 255, 65, 0.55);
  font-family: var(--font);
  font-weight: 700;
  font-size: 11px;
  cursor: pointer;
  transition: all 0.15s;

  &:hover:not(:disabled) {
    border-color: var(--green);
    color: var(--green);
    background: rgba(0, 255, 65, 0.07);
  }

  &--active {
    border-color: var(--green) !important;
    background: rgba(0, 255, 65, 0.15) !important;
    color: var(--green) !important;
    box-shadow: 0 0 6px var(--green-glow);
  }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

// ── 提交按钮 ──────────────────────────────────────────────────────────────
.form-footer {
  display: flex;
  justify-content: center;
  padding-top: 4px;
}

.submit-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 160px;
  padding: 10px 28px;
  background: var(--green);
  border: none;
  border-radius: var(--radius);
  color: #000000;
  font-family: var(--font);
  font-weight: 700;
  font-size: 14px;
  letter-spacing: 0.1em;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 0 14px var(--green-glow);

  &:hover:not(:disabled) {
    background: #33ff66;
    box-shadow: 0 0 22px rgba(0, 255, 65, 0.6);
    transform: translateY(-1px);
  }

  &:active:not(:disabled) {
    transform: translateY(0);
  }

  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
    box-shadow: none;
  }
}

.btn-inner {
  display: flex;
  align-items: center;
  gap: 6px;
}

// ── 旋转动画 ──────────────────────────────────────────────────────────────
.spinner {
  display: inline-block;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to   { transform: rotate(360deg); }
}
</style>
