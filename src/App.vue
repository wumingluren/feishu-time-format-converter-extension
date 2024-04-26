<template>
  <main>
    <!-- <h4>编辑 <code>src/App.vue</code> 并保存以重新加载</h4> -->
    <p style="margin-bottom: 1rem">
      使用说明：<a
        href="https://h1pyhdrylh8.feishu.cn/docx/KuGLdfgwDo4RwTxWHkQcjuZwnug?from=from_copylink"
        >帮助文档</a
      >
      开源链接：<a
        href="https://github.com/wumingluren/feishu-time-format-converter-extension"
        >github</a
      >
    </p>

    <Form />
  </main>
</template>

<script setup>
import Form from "./components/Form.vue";
</script>

<style scoped>
main {
  padding: 1rem;
}
h4 {
  font-size: calc(1.275rem + 0.3vw);
  margin-bottom: 1rem;
}
code {
  font-size: 0.875em;
  color: #d63384;
  word-wrap: break-word;
}
</style>
