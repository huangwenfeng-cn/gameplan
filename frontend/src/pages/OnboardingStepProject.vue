<template>
  <FormControl
    label="项目名称"
    placeholder="产品发布"
    @update:modelValue="(project) => $emit('update:modelValue', { ...modelValue, project })"
    v-model="modelValue.project"
  />
</template>
<script>
export default {
  name: 'OnboardingStepProject',
  props: ['modelValue'],
  emits: ['update:modelValue'],
}
</script>
