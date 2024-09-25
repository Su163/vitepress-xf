<template>
  <div>
    <h1>FormConfig 生成器</h1>

    <!-- 使用 a-row 和 a-col 创建两列布局 -->
    <a-row :gutter="24">
      <!-- 左侧输入区 -->
      <a-col :span="12">
        <a-form layout="vertical">
          <!-- 输入框，用户输入表单 HTML -->
          <a-form-item label="输入表单 HTML">
            <a-textarea
                v-model:value="inputHtml"
                :rows="15"
                placeholder="请输入表单 HTML 代码"
            />
          </a-form-item>

          <!-- 按钮：解析并生成 FormConfig -->
          <a-button type="primary" @click="generateFormConfig">生成 FormConfig</a-button>
        </a-form>
      </a-col>

      <!-- 右侧输出区，仅展示 FormConfig -->
      <a-col :span="12">
        <a-form layout="vertical">
          <!-- 显示生成的 FormConfig -->
          <a-form-item label="生成的 FormConfig">
            <a-textarea v-model:value="formConfigText" :rows="15" readonly />
          </a-form-item>

          <!-- 复制 FormConfig 按钮 -->
          <a-button type="primary" @click="copyFormConfig" :disabled="formConfig.length === 0">复制 FormConfig</a-button>
        </a-form>
      </a-col>
    </a-row>
  </div>
</template>

<script setup>
// 引入 Vue 组合式 API
import { ref } from 'vue';
// import DynamicForm from './DynamicForm.vue'; // 引入封装的动态表单组件

// 输入的表单 HTML 代码
const inputHtml = ref('');

// 生成的 formConfig（用于动态表单组件）
const formConfig = ref([]);

// 生成的 formConfig 字符串，用于右侧展示
const formConfigText = ref('');

// 查询参数结果，提交表单后展示
const queryResult = ref(null);

// 查询方法
const searchQuery = (formData) => {
  console.log('查询参数:', formData);
  queryResult.value = formData; // 将查询结果保存到 queryResult 中展示
};

// 重置方法
const resetForm = () => {
  console.log('表单已重置');
  queryResult.value = null; // 重置查询结果
};

// 扩展的控件类型映射关系，添加对 j-search-select-tag、j-date 和 j-date-custom 的支持
const controlTypeMap = {
  'a-input': 'input',
  'j-input': 'input',
  'a-select': 'select',
  'j-search-select-tag': 'search-select',
  'j-date': 'date',
  'j-date-custom': 'date-custom',  // 支持 j-date-custom
  'j-dict-select-tag': 'dict-select',
  'j-multi-select-tag': 'multi-select',
};

// 递归解析 HTML 节点，包括自定义标签 <template>
const parseFormItems = (parentNode) => {
  const items = [];

  // 遍历子节点
  parentNode.childNodes.forEach((child) => {
    // 如果是 <template> 标签，递归处理其内部内容
    if (child.tagName === 'TEMPLATE') {
      items.push(...parseFormItems(child.content));
    }

    // 如果是 <a-form-item>，处理该节点
    else if (child.tagName === 'A-FORM-ITEM') {
      const label = child.getAttribute('label') || '未命名字段';

      // 查找表单控件，包括自定义的 <j-search-select-tag>、<j-date>、<j-date-custom>
      const inputElement = child.querySelector('a-input, j-input, j-search-select-tag, j-dict-select-tag, a-select, j-date, j-date-custom, j-multi-select-tag');
      if (!inputElement) return;

      let tagName = inputElement.tagName.toLowerCase();
      let type = controlTypeMap[tagName] || 'input';  // 从映射关系中获取类型，默认为 'input'

      // 获取 v-model 绑定字段，并去掉前缀 'queryParam.'
      let vModel = inputElement.getAttribute('v-model');
      if (vModel && vModel.startsWith('queryParam.')) {
        vModel = vModel.replace('queryParam.', '');
      }

      // 获取其他属性，如 placeholder
      const placeholder = inputElement.getAttribute('placeholder') || '';

      // 处理字典选择器的 dict 和 dictCode
      const dict = inputElement.getAttribute('dict') || null;
      const dictCode = inputElement.getAttribute('dictCode') || null;

      // 生成 formConfig 项
      const formItem = {
        label: label,
        type: type,
        model: vModel,
        props: {
          placeholder: placeholder,
        },
        colSpan: 6, // 默认列宽
      };

      // 如果存在 dict，添加到 formItem
      if (dict) {
        formItem.dict = dict;
      }

      // 如果存在 dictCode，添加到 formItem
      if (dictCode) {
        formItem.dictCode = dictCode;
      }

      items.push(formItem);
    }

    // 处理 <a-col> 及其子节点
    else if (child.tagName === 'A-COL') {
      items.push(...parseFormItems(child));
    }
  });

  return items;
};

// 解析 HTML，生成 formConfig
const generateFormConfig = () => {
  // 创建一个 div 用于解析 HTML 字符串
  const parser = new DOMParser();
  console.log('parser',parser);
  const doc = parser.parseFromString(inputHtml.value, 'text/html');

  // 获取 a-row 元素作为起点
  const formRows = doc.querySelectorAll('a-row');

  // 遍历 a-row，解析所有子元素
  const config = Array.from(formRows).flatMap(parseFormItems);

  // 生成的 config 赋值给 formConfig
  formConfig.value = config;

  // 将生成的 formConfig 转换为字符串并赋值给 formConfigText
  formConfigText.value = JSON.stringify(config, null, 2);
};

// 复制 FormConfig 到剪贴板
const copyFormConfig = async () => {
  if (formConfigText.value) {
    try {
      await navigator.clipboard.writeText(formConfigText.value);
      alert('FormConfig 已复制到剪贴板');
    } catch (error) {
      alert('复制失败，请手动复制');
    }
  }
};
</script>

<style scoped>
h1 {
  margin-bottom: 20px;
}
</style>
