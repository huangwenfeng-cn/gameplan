<template>
  <div class="space-y-4">
    <FormControl
      v-for="(email, index) in modelValue.emails"
      :key="index"
      label="邮箱"
      placeholder="zhang@example.com"
      :modelValue="email"
      @update:modelValue="
        (email) =>
          $emit('update:modelValue', {
            ...modelValue,
            emails: modelValue.emails.map((e, i) => (i === index ? email : e)),
          })
      "
    />
    <Button
      v-show="modelValue.emails.length <= 6"
      iconLeft="plus"
      @click="
        $emit('update:modelValue', {
          ...modelValue,
          emails: [...modelValue.emails, ''],
        })
      "
    >
      添加另一个邮箱
    </Button>
  </div>
</template>
<script>
export default {
  name: 'OnboardingStepProject',
  props: ['modelValue'],
  emits: ['update:modelValue'],
}
</script>
