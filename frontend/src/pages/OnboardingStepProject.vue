<template>
  <FormControl
    label="项目名称"
    placeholder="Product Launch"
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
