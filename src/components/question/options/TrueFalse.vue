<script setup lang="ts">
import { onMounted, ref, watch } from 'vue';
import type { TrueFalseQuestion } from '../../../types/question';

const props = defineProps<{
  question: TrueFalseQuestion;
  userAnswer: boolean | undefined,
  isSubmit: boolean
}>();

const selected = ref<boolean>();
function setUserAnswer() {
  // 有用户选项时填充
  if (props.userAnswer !== undefined) {
    selected.value = props.userAnswer;
  } else {
    selected.value = undefined;
  }
}
// 重新挂载时
onMounted(setUserAnswer);
// 发生变更时
watch(() => props.userAnswer, setUserAnswer);

const emit = defineEmits(['answer']);

const submitAnswer = () => {
  if (selected.value !== undefined) {
    emit('answer', selected.value);
  }
};

const getIcon = (option: boolean) => {
  if (props.isSubmit) {
    // 是正确选项，用户选了
    if (option === props.question.answer) {
      // 用户没选的不管
      return props.userAnswer !== undefined && option === props.userAnswer ? 'success' : undefined;
    }
    // 用户选了但不是正确选项
    if (props.userAnswer !== undefined && option === props.userAnswer) {
      return 'error';
    }
  }

  return undefined;
};
</script>

<template>
  <el-radio-group v-model="selected" :onchange="submitAnswer">
    <el-radio :value="undefined" disabled>请选择</el-radio>
    <div>
      <div class="option" :class="props.isSubmit && props.question.answer === true ? 'option-success' : ''">
        <el-radio :value="true" :disabled="isSubmit">A. 正确</el-radio>
      </div>

      <el-result v-if="getIcon(true)" :icon="getIcon(true)" />
    </div>
    <div>
      <div class="option" :class="props.isSubmit && props.question.answer === false ? 'option-success' : ''">
        <el-radio :value="false" :disabled="isSubmit">B. 错误</el-radio>
      </div>

      <el-result v-if="getIcon(false)" :icon="getIcon(false)" />
    </div>

  </el-radio-group>
</template>
