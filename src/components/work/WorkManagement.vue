<script setup>
import { computed, onBeforeUnmount, ref, watch } from 'vue'
import { useStore } from 'vuex'
import { ElMessage, ElMessageBox } from 'element-plus'
import { listWork, saveWork, removeWork, getPeople } from '../../api/work'
import { workModules, dateTime } from './workModel'
import WorkFormDialog from './WorkFormDialog.vue'
import WorkTable from './WorkTable.vue'
import MarkdownContent from '../common/MarkdownContent.vue'
const props = defineProps({ resource: { type: String, required: true } })
const store = useStore(), config = computed(() => workModules[props.resource])
const items = ref([]), page = ref(1), total = ref(0), keyword = ref(''), status = ref(''), applied = ref(''), loading = ref(false), saving = ref(false), error = ref('')
const editor = ref(false), editing = ref(null), detail = ref(null), people = ref([])
const busy = computed(() => loading.value || saving.value)
let generation = 0
const can = permission => store.getters.can(permission)
const own = row => props.resource !== 'reports' || row.userId === store.state.userInfo?.id
const canEdit = row => can(config.value.edit) && own(row) && (props.resource !== 'tasks' || can('task:manage') || [row.createdBy,row.assigneeId].includes(store.state.userInfo?.id))
const canRemove = row => can(config.value.remove) && own(row)
async function load(reset = false) {
  const request = ++generation, resource = props.resource
  loading.value = true; error.value = ''
  if (reset) page.value = 1
  try {
    let result = await listWork(resource, { page: page.value, keyword: applied.value, status: status.value })
    if (request !== generation) return
    if (result.page > result.totalPages) result = await listWork(resource, { page: result.totalPages, keyword: applied.value, status: status.value })
    if (request !== generation) return
    items.value = result.items; total.value = result.total; page.value = result.page
  } catch (failure) { if (request === generation) error.value = failure.userMessage || '加载失败，请重试' }
  finally { if (request === generation) loading.value = false }
}
async function open(row = null) {
  if (busy.value || !(row ? canEdit(row) : can(config.value.create))) return
  if (props.resource === 'tasks') {
    loading.value = true
    try { people.value = await getPeople() } catch { return } finally { loading.value = false }
  }
  editing.value = row ? { ...row } : null; editor.value = true
}
async function save(body) {
  if (busy.value || !(editing.value ? canEdit(editing.value) : can(config.value.create))) return
  saving.value = true
  try { await saveWork(props.resource, editing.value?.id, body); editor.value = false; ElMessage.success('保存成功'); await load() }
  catch { /* API displays errors and retains form input. */ }
  finally { saving.value = false }
}
async function remove(row) {
  if (busy.value || !canRemove(row)) return
  saving.value = true
  try {
    await ElMessageBox.confirm(`确定删除此${config.value.singular}？`, '删除确认', { type: 'warning', confirmButtonText: '删除', cancelButtonText: '取消' })
    await removeWork(props.resource, row.id); ElMessage.success('已删除'); await load()
  } catch { /* Cancellation and API failure leave the list intact. */ }
  finally { saving.value = false }
}
onBeforeUnmount(() => ++generation)
watch(() => props.resource, () => { editor.value = false; detail.value = null; keyword.value = ''; applied.value = ''; status.value = ''; load(true) }, { immediate: true })
</script>
<template>
  <el-card class="work-management" shadow="never">
    <div class="work-heading"><div><h1 class="page-title">{{ config.title }}</h1><p class="page-description">{{ config.description }}</p></div><el-button v-if="can(config.create)" type="primary" :disabled="busy" @click="open()">新增{{ config.singular }}</el-button></div>
    <div class="work-filters"><el-input v-model="keyword" clearable placeholder="输入关键词搜索" :disabled="busy" @keyup.enter="applied = keyword; load(true)" /><el-select v-if="config.statuses" v-model="status" clearable placeholder="全部状态" :disabled="busy" @change="load(true)"><el-option v-for="option in config.statuses" :key="option.value" v-bind="option" /></el-select><el-button :disabled="busy" @click="applied = keyword; load(true)">搜索</el-button><el-button :disabled="busy" @click="keyword = ''; applied = ''; status = ''; load(true)">重置</el-button><el-button :disabled="busy" @click="load()">刷新</el-button></div>
    <el-alert v-if="error" :title="error" type="error" :closable="false" show-icon />
    <WorkTable :config="config" :items="items" :busy="busy" :can-edit="canEdit" :can-remove="canRemove" @detail="detail = $event" @edit="open" @remove="remove" />
    <el-pagination v-model:current-page="page" :total="total" :page-size="20" layout="prev, pager, next, total" :disabled="busy" @current-change="load()" />
    <WorkFormDialog v-model="editor" :config="config" :record="editing" :people="people" :busy="saving" @save="save" />
    <el-dialog :model-value="!!detail" :title="`${config.singular}详情`" width="min(760px, 94vw)" @update:model-value="!$event && (detail = null)">
      <div v-if="detail" class="work-detail"><template v-for="field in config.fields" :key="field.key"><h3>{{ field.label }}</h3><MarkdownContent v-if="field.type === 'textarea'" :content="detail[field.key] || '暂无内容'" /><p v-else>{{ field.people ? (detail.assigneeName || detail.assigneeAccount || '未分配') : field.options ? field.options.find(o => o.value === detail[field.key])?.label : field.type === 'switch' ? detail[field.key] ? '是' : '否' : detail[field.key] || '—' }}</p></template><small>更新于 {{ dateTime(detail.updatedAt) }}</small></div>
    </el-dialog>
  </el-card>
</template>
<style scoped>
.work-heading { display:flex; justify-content:space-between; gap:16px; align-items:center; }
.work-filters { display:flex; gap:10px; margin:22px 0; flex-wrap:wrap; }
.work-filters .el-input { width:260px; }.work-filters .el-select { width:140px; }
.el-pagination { margin-top:22px; justify-content:flex-end; max-width:100%; overflow:auto; }
.work-detail h3 { margin:22px 0 8px; font-size:13px; color:#7a8a9e; }.work-detail p { white-space:pre-wrap; overflow-wrap:anywhere; }.work-detail small { display:block; margin-top:25px; color:#91a0b2; }
@media(max-width:600px) { .work-heading { align-items:flex-start; }.work-filters .el-input { width:100%; }.work-heading .el-button { flex-shrink:0; } }
</style>
