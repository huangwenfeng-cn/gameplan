<template>
  <FormControl
    label="团队名称"
    placeholder="市场营销部"
    @update:modelValue="(team) => $emit('update:modelValue', { ...modelValue, team })"
    v-model="modelValue.team"
  />
</template>
<script>
export default {
  name: 'OnboardingStepTeam',
  props: ['modelValue'],
  emits: ['update:modelValue'],
}
</script>
