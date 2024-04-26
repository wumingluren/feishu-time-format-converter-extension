<template>
  <el-form
    ref="form"
    class="form"
    :model="formData"
    :rules="rules"
    label-position="top"
  >
    <el-form-item label="选择模式" size="large" prop="selectModel">
      <el-radio-group v-model="formData.selectModel">
        <el-radio-button :value="1">单元格</el-radio-button>
        <el-radio-button :value="2">字段</el-radio-button>
      </el-radio-group>
    </el-form-item>
    <el-form-item label="转换后的格式" size="large" prop="format">
      <p>默认为：年-月-日-时-分-秒</p>
      <el-input
        v-model="formData.format"
        placeholder="请输入转换后的格式"
        clearable
      />
    </el-form-item>
    <el-form-item
      label="选择字段"
      size="large"
      prop="fieldId"
      v-if="formData.selectModel === 2"
    >
      <el-select
        v-model="formData.fieldId"
        placeholder="请选择字段"
        style="width: 100%"
      >
        <el-option
          v-for="item in fieldList"
          :key="item.id"
          :label="item.name"
          :value="item.id"
        />
      </el-select>
    </el-form-item>

    <el-form-item>
      <el-button
        type="primary"
        size="large"
        @click="confirm(form)"
        :loading="isLoading"
        style="width: 100%"
        >执行转换</el-button
      >
    </el-form-item>
  </el-form>
</template>

<script setup>
import { bitable } from "@lark-base-open/js-sdk";
import { ref, reactive, onMounted, watch } from "vue";
import {
  ElMessage,
  ElMessageBox,
  ElButton,
  ElForm,
  ElFormItem,
  ElSelect,
  ElOption,
  ElUpload,
} from "element-plus";
import { getTimeNewValue } from "../utils.js";

const formData = ref({
  selectModel: 1, // 选择模式 1 单元格 2 字段
  fieldId: "",
  format: "YYYY-MM-DD HH:mm:ss",
});
const form = ref();
const rules = reactive({
  selectModel: [
    {
      required: true,
      message: "请选择模式",
      trigger: "blur",
    },
  ],
  fieldId: [
    {
      required: true,
      message: "请选择字段",
      trigger: "blur",
    },
  ],
  format: [
    {
      required: true,
      message: "请输入转换后的格式",
      trigger: "blur",
    },
  ],
});

onMounted(async () => {});

// 数据表id
const databaseId = ref();
// 视图id
const viewId = ref();

// 当前选中字段id
const currentFieldId = ref();
const recordId = ref();
// 当前选中数据
const currentValue = ref();

// 监听当前选中（数据表、单元格、视图）改变事件
bitable.base.onSelectionChange(async (event) => {
  console.log("监听当前选中改变事件", event);
  // 获取点击的字段id和记录id
  currentFieldId.value = event.data.fieldId;
  recordId.value = event.data.recordId;

  // 获取当前数据表和视图
  databaseId.value = event.data.tableId;
  viewId.value = event.data.viewId;

  // 获取当前选中的数据表
  const table = await bitable.base.getActiveTable();
  if (currentFieldId.value && recordId.value) {
    // getCellValue 获取指定单元格的取值。
    // 修改当前选中数据值
    let data = await table.getCellValue(currentFieldId.value, recordId.value);
    if (data && data[0]?.text !== currentValue.value) {
      currentValue.value = data[0].text;
    }
  }
});

// 切换选择模式时,重置选择
watch(
  () => formData.value.selectModel,
  async (newValue, oldValue) => {
    console.log("xxxxxxxxxxxxxxx");
    if (newValue === 1) return;
    // 单列和数据表模式，默认选中当前数据表和当前视图

    if (newValue === 2) {
      // 读取当前 table id, field id
      const selection = await bitable.base.getSelection();
      databaseId.value = selection.tableId;
      formData.value.fieldId = "";
      fieldList.value = [];

      viewId.value = selection.viewId;
      console.log("查看视图", selection);
    }
  }
);

// 字段列表
const fieldList = ref();
// 根据视图id获取字段列表
watch(viewId, async (newValue, oldValue) => {
  const table = await bitable.base.getTable(databaseId.value);
  const view = await table.getViewById(newValue);
  const _list = await view.getFieldMetaList();
  console.log("字段有序列表", _list);
  // 只展示文本相关字段
  fieldList.value = _list.filter((item) => item.type === 1);
});

const isLoading = ref(false);
// 执行转换
const confirm = async (formEl) => {
  if (!formEl) return;

  try {
    // 尝试验证表单
    await formEl.validate();
    // 如果没有抛出错误，说明验证通过，继续执行提交逻辑
    console.log("submit!");
    // 在这里编写表单提交的代码
  } catch (errors) {
    // 如果在验证过程中抛出了错误，说明验证失败
    console.error("error submit!", errors);
    // 打印或处理错误信息
    return;
  }

  isLoading.value = true;

  if (formData.value.selectModel === 1) {
    if (currentFieldId.value && recordId.value) {
      await cellModel();
    } else {
      ElMessage({
        type: "error",
        message: "请选择需要转换的单元格!",
        duration: 1500,
        showClose: true,
      });
    }
  } else {
    await fieldModel();
  }

  isLoading.value = false;
};

// 转换单元格数据
async function cellModel() {
  // 检查外部变量有效性
  if (!currentFieldId.value || !recordId.value) {
    ElMessage({
      message: "当前字段ID或记录ID不存在",
      type: "error",
      duration: 1500,
    });
    return;
  }

  try {
    const table = await bitable.base.getActiveTable();

    // 检查table对象及其setCellValue方法的有效性
    if (!table || typeof table.setCellValue !== "function") {
      throw new Error("无效的表格对象或setCellValue方法");
    }

    let newValue = getTimeNewValue(currentValue.value);
    if (!newValue) {
      ElMessage({
        message: "当前数据无法进行转换",
        type: "warning",
        duration: 1500,
      });
      return;
    }
    // 修改单元格内容
    await table.setCellValue(currentFieldId.value, recordId.value, newValue);

    ElMessage({
      message: "数据转换成功!",
      type: "success",
      duration: 1500,
    });
  } catch (error) {
    // 异常处理，展示错误信息
    ElMessage({
      message: error.message,
      type: "error",
      duration: 1500,
    });
  }
}

// 转换整列数据
async function fieldModel() {
  ElMessage({
    message: "开始转换数据~",
    type: "success",
    duration: 1500,
  });

  const table = await bitable.base.getTable(databaseId.value);
  // 获取当前的记录列表
  const recordList = await table.getRecordList();
  // 获取所有记录 id
  const recordIds = await table.getRecordIdList();

  let _list = [];

  for (const record of recordList) {
    const id = record.id;
    // 获取索引
    const index = recordList.recordIdList.findIndex((iId) => iId === id);

    // 根据字段id获取字段
    const field = await table.getFieldById(formData.value.fieldId);
    // 获取单元格
    const cell = await field.getCell(recordIds[index]);
    // 获取指定单元格的值
    const val = await cell.getValue();

    if (!val) continue;

    let newValue = getTimeNewValue(val[0]?.text);

    if (newValue) {
      // 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [formData.value.fieldId]: newValue,
        },
      });
    } else {
      console.log("跳过");
    }
  }

  // 此处一次性全部替换 修改多条记录，单次上限 5000 条
  await table.setRecords(_list);

  ElMessage({
    message: "数据转换结束!",
    type: "success",
    duration: 1500,
  });
}
</script>

<style lang="less" scoped>
.form :deep(.el-form-item__label) {
  font-size: 16px;
  color: var(--el-text-color-primary);
  margin-bottom: 0;
}
.form :deep(.el-form-item__content) {
  font-size: 16px;
}
.upload-box {
  width: 100%;
}

:deep(.el-upload-dragger) {
  padding: 20px;
}
</style>
