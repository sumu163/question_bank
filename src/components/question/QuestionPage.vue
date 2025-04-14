<script setup lang="ts">
import { ref, computed, watch } from 'vue';
import SingleChoice from './options/SingleChoice.vue';
import MultipleChoice from './options/MultipleChoice.vue';
import TrueFalse from './options/TrueFalse.vue';
import Sorting from './options/SortingProblem.vue';
import type { Question, SingleChoiceQuestion, MultipleChoiceQuestion, TrueFalseQuestion, SortingQuestion, Answer, SubQuestion, QuestionType } from '../../types/question';
import { ElMessage, ElMessageBox } from 'element-plus';
import SearchQuestion from './SearchQuestion.vue';

const props = defineProps<{
  questions: Question[];
  // 是否显示工具？默认不显示
  showSearchTools?: boolean;
}>();

// 当前题库
const currentQuestions = ref<Question[]>(props.questions);
// 监听questions的改变
watch(() => props.questions, (newVal) => {
  currentQuestions.value = newVal;
  userAnswers.value = [];
  isSubmitted.value = [];
  currentIndex.value = 0;
});

const currentIndex = ref(0);
const userAnswers = ref<Answer[] | Answer[][]>([]);

const currentQuestion = computed(() => currentQuestions.value[currentIndex.value]);

const handleAnswer = (answer: Answer, index?: number) => {
  if (index === undefined) {
    // 不是小题，直接给答案。
    userAnswers.value[currentIndex.value] = answer;
  } else {
    if (!userAnswers.value[currentIndex.value]) {
      userAnswers.value[currentIndex.value] = [];
    }
    const subAns = userAnswers.value[currentIndex.value] as Answer[];
    subAns[index] = answer;
  }
};

// 获取用户选的选项
const getUserAnswer = (index?: number): Answer | undefined => {
  if (index === undefined) {
    return userAnswers.value[currentIndex.value] as Answer | undefined;
  } else {
    const subAns = userAnswers.value[currentIndex.value] as Answer[] | undefined;
    return subAns ? subAns[index] : subAns;
  }
};

let changeQuestionTimer: ReturnType<typeof setTimeout>;
const nextQuestion = (time = 0) => {
  clearTimeout(changeQuestionTimer);
  if (time > 0) {
    changeQuestionTimer = setTimeout(() => {
      if (currentIndex.value < currentQuestions.value.length - 1) {
        currentIndex.value++;
      }
    }, time);
  } else if (currentIndex.value < currentQuestions.value.length - 1) {
    currentIndex.value++;
  }
};

const prevQuestion = (time = 0) => {
  clearTimeout(changeQuestionTimer);
  if (time > 0) {
    changeQuestionTimer = setTimeout(() => {
      if (currentIndex.value > 0) {
        currentIndex.value--;
      }
    }, time);
  } else if (currentIndex.value > 0) {
    currentIndex.value--;
  }
};

// 绑定键盘事件
document.addEventListener('keydown', (e) => {
  if (e.key === 'ArrowLeft') {
    prevQuestion();
  } else if (e.key === 'ArrowRight') {
    nextQuestion();
  }
});

// 记录用户是否提交
const isSubmitted = ref<boolean[]>([]);

// 用户提交事件
const toSubmit = async () => {
  const confirmSubmit = () => ElMessageBox.confirm(
    '有未回答的题目，确定提交吗?',
    'Warning',
    {
      confirmButtonText: 'OK',
      cancelButtonText: 'Cancel',
      type: 'warning',
    }
  )
    .then(() => {
      return true;
    })
    .catch(() => {
      ElMessage({
        type: 'info',
        message: '取消提交！',
      });
      return false;
    });
  // 检查是否有未答的题
  const question = currentQuestions.value[currentIndex.value];
  let submit = true;
  if (question.type === 'hasSubQuestion') {
    const subQuestions = question.options as SubQuestion[];
    for (let i = 0; i < subQuestions.length; i++) {
      if (getUserAnswer(i) === undefined) {
        submit = await confirmSubmit();
        break;
      }
    }
  } else {
    if (getUserAnswer() === undefined) {
      submit = await confirmSubmit();
    }
  }
  if (submit) {
    isSubmitted.value[currentIndex.value] = true;
    // 如果回答正确，调到下一道题
    // 比对答案的方法
    const isTrue = (type: QuestionType, userAns: Answer, ans: Answer) => {
      if (userAns === undefined) {
        return false;
      }
      if (type === 'multiple') {
        return (userAns as string[]).sort().join('') === (ans as string[]).sort().join('');
      } else if (type === 'sorting') {
        return (userAns as string[]).join('') === (ans as string[]).join('');
      } else {
        // 单选、判断
        return userAns === ans;
      }
    }

    if (question.type === 'hasSubQuestion') {
      const subQuestions = question.options as SubQuestion[];
      for (let i = 0; i < subQuestions.length; i++) {
        if (!getUserAnswer(i) || !isTrue(subQuestions[i].type, getUserAnswer(i) as Answer, subQuestions[i].answer)) {
          markWrong();
          ElMessage({
            type: 'error',
            message: '回答错误！',
          });
          return;
        }
      }
    } else {
      if (userAnswers.value[currentIndex.value] === undefined || !isTrue(question.type, userAnswers.value[currentIndex.value] as Answer, question.answer as Answer)) {
        markWrong();
        ElMessage({
          type: 'error',
          message: '回答错误！',
        });
        return;
      }
    }
    ElMessage({
      type: 'success',
      message: '回答正确！',
    });

    // 下一题
    nextQuestion(1000);
  }
}

// 记录回答错误的题
const wrongQuestionsIndex = ref<number[]>([]);
const markWrong = () => {
  if (!wrongQuestionsIndex.value.includes(currentIndex.value)) {
    wrongQuestionsIndex.value.push(currentIndex.value);
  }
}

// 重置当前的回答
const resetAnswer = () => {
  currentIndex.value = 0;
  userAnswers.value = [];
  isSubmitted.value = [];
  wrongQuestionsIndex.value = [];
}
// 检查用户题是否已经打完？
const getUnAnswerQuestionIndex = (): number[] => {
  const res = [] as number[];
  for (let i = 0; i < currentQuestions.value.length; i++) {
    if (isSubmitted.value[i] === undefined) {
      res.push(i);
    }
  }
  return res;
}
// 回答错题、回答所有题
const toErrorQuestion = () => {
  const unAnswerQuestionIndex = getUnAnswerQuestionIndex();
  const msg = unAnswerQuestionIndex.length > 0 ? `有未回答的题目:【${unAnswerQuestionIndex.map(i => i + 1).join(',')}】，确定切换吗?` : '没有未回答的题目，确定切换吗?';
  ElMessageBox.confirm(
    msg,
    'Warning',
    {
      confirmButtonText: 'OK',
      cancelButtonText: 'Cancel',
      type: 'warning',
    }
  )
    .then(() => {
      currentQuestions.value = wrongQuestionsIndex.value.map((index) => currentQuestions.value[index]);
      resetAnswer();
    })
    .catch(() => {
      ElMessage({
        type: 'info',
        message: '取消切换！',
      });
    });

}
const toAllQuestion = () => {
  const unAnswerQuestionIndex = getUnAnswerQuestionIndex();

  const msg = unAnswerQuestionIndex.length > 0 ? `有未回答的题目:【${unAnswerQuestionIndex.map(i => i + 1).join(',')}】，确定切换吗?` : '没有未回答的题目，确定切换吗?';
  ElMessageBox.confirm(
    msg,
    'Warning',
    {
      confirmButtonText: 'OK',
      cancelButtonText: 'Cancel',
      type: 'warning',
    }
  )
    .then(() => {
      currentQuestions.value = props.questions;
      resetAnswer();
    })
    .catch(() => {
      ElMessage({
        type: 'info',
        message: '取消切换！',
      });
    });

}

// 题号
const questionNo = ref<number>(1);

const handleQuestionNoChange = () => {
  const no = questionNo.value;
  if (no > 0 && no <= currentQuestions.value.length) {
    currentIndex.value = no - 1;
  } else {
    questionNo.value = 1;
  }
}

// 更多工具是否展开
const moreTools = ref("");
// 搜索题目相关参数
// 搜索参数
const searchContent = ref(''); // 默认搜索内容
const searchField = ref<'title' | 'options' | "all">(); // 默认搜索题目
const useRegex = ref(false); // 默认不启用正则

</script>

<template>
  <div class="question-page">
    <!-- 进度条 -->
    <div><el-progress :percentage="currentIndex / currentQuestions.length"
        :format="() => `${currentIndex + 1} / ${currentQuestions.length}`" /></div>
    <!-- 题目 -->
    <h3 class="title">
      <div>
        {{ currentQuestion.title.trim() }}
      </div>
    </h3>
    <!-- 选项相关 -->
    <div class="options">
      <SingleChoice v-if="currentQuestion.type === 'single'" :question="currentQuestion as SingleChoiceQuestion"
        :userAnswer="getUserAnswer() as (string | undefined)" @answer="handleAnswer"
        :isSubmit="!!isSubmitted[currentIndex]" />
      <MultipleChoice v-else-if="currentQuestion.type === 'multiple'"
        :userAnswer="getUserAnswer() as (string[] | undefined)" :question="currentQuestion as MultipleChoiceQuestion"
        @answer="handleAnswer" :isSubmit="!!isSubmitted[currentIndex]" />
      <TrueFalse v-else-if="currentQuestion.type === 'truefalse'" :question="currentQuestion as TrueFalseQuestion"
        :userAnswer="getUserAnswer() as (boolean | undefined)" @answer="handleAnswer"
        :isSubmit="!!isSubmitted[currentIndex]" />
      <Sorting v-else-if="currentQuestion.type === 'sorting'" :question="currentQuestion as SortingQuestion"
        :userAnswer="getUserAnswer() as (string[] | undefined)" @answer="handleAnswer"
        :isSubmit="!!isSubmitted[currentIndex]" />
      <!-- 有多个小题 -->
      <div v-else-if="currentQuestion.type === 'hasSubQuestion'">
        <div v-for="(sub, index) in (currentQuestion.options as SubQuestion[])" :key="index">
          <h4>{{ sub.title }}</h4>
          <SingleChoice v-if="sub.type === 'single'"
            :question="(currentQuestion.options as SubQuestion[])[index] as SingleChoiceQuestion"
            :userAnswer="getUserAnswer(index) as (string | undefined)" @answer="(ans) => handleAnswer(ans, index)"
            :isSubmit="!!isSubmitted[currentIndex]" />
          <MultipleChoice v-else-if="sub.type === 'multiple'"
            :question="(currentQuestion.options as SubQuestion[])[index] as MultipleChoiceQuestion"
            :userAnswer="getUserAnswer(index) as (string[] | undefined)" @answer="(ans) => handleAnswer(ans, index)"
            :isSubmit="!!isSubmitted[currentIndex]" />
          <TrueFalse v-else-if="sub.type === 'truefalse'"
            :question="(currentQuestion.options as SubQuestion[])[index] as TrueFalseQuestion"
            :userAnswer="getUserAnswer(index) as (boolean | undefined)" @answer="(ans) => handleAnswer(ans, index)"
            :isSubmit="!!isSubmitted[currentIndex]" />
          <Sorting v-else-if="sub.type === 'sorting'"
            :question="(currentQuestion.options as SubQuestion[])[index] as SortingQuestion"
            :userAnswer="getUserAnswer(index) as (string[] | undefined)" @answer="(ans) => handleAnswer(ans, index)"
            :isSubmit="!!isSubmitted[currentIndex]" />
        </div>
      </div>


    </div>

    <!-- 操作按钮 -->
    <div class="operate common-operate">
      <el-button @click="prevQuestion" :disabled="currentIndex === 0">上一题</el-button>
      <el-button @click="toSubmit" :disabled="isSubmitted[currentIndex]">提交</el-button>
      <el-button @click="nextQuestion" :disabled="currentIndex === currentQuestions.length - 1">下一题</el-button>

    </div>

    <el-collapse class="more-tools" v-model="moreTools">
      <el-collapse-item title="更多" name="1">
        <div class="more-tools-operate">
          <!-- 回答错题 -->
          <div>
            <el-button @click="toErrorQuestion" :disabled="wrongQuestionsIndex.length === 0">回答错题</el-button>
            <el-button @click="toAllQuestion">回答所有题</el-button>
          </div>

          <div>
            <label for="questionNo">题号：</label>
            <el-input-number id="questionNo" v-model="questionNo" :min="1" :max="currentQuestions.length"
              @change="handleQuestionNoChange" />
          </div>

          <div class="search" v-if="showSearchTools">
            <div class="search-param">
              <h3>题目搜索:</h3>
              <el-input v-model="searchContent" placeholder="请输入搜索内容" style="width: 200px; margin-right: 10px" />
              <el-select v-model="searchField" placeholder="搜索范围" style="width: 120px; margin-right: 10px">
                <el-option label="题目" value="title" />
                <el-option label="选项" value="options" />
                <el-option label="All" value="all" />
              </el-select>
              <el-checkbox v-model="useRegex" label="启用正则" />
              <SearchPage :questions="questions" :search-content="searchContent" :search-field="searchField"
                :use-regex="useRegex" />
            </div>

            <SearchQuestion v-if="moreTools && searchContent?.trim()" :questions="currentQuestions"
              :search-content="searchContent" :search-field="searchField" :use-regex="useRegex" />
          </div>
        </div>
      </el-collapse-item>
    </el-collapse>

  </div>


</template>
<style scoped>
.question-page {
  display: flex;
  flex-direction: column;
  height: 100%;

  .title {
    flex: 1;
    min-height: 14px;
    white-space: pre-wrap;

    & div {
      border: 1px solid #bbb;
      padding: 5px;
      border-radius: 5px;
    }
  }

  .operate {
    margin-top: 20px;
    text-align: center;

    display: flex;
    flex-direction: row;
    justify-content: center;
    gap: 5px;

    flex-wrap: wrap;
  }

  .more-tools {
    margin-top: 5px;

    .more-tools-operate {
      display: flex;
      flex-direction: column;
      gap: 5px;

      &>div {
        border: 1px solid #bbb;
        border-radius: 5px;
        padding: 5px;
      }

      .search {
        display: flex;
        flex-direction: column;
        gap: 5px;

        .search-param {
          display: flex;
          flex-direction: row;
          gap: 5px;
          flex-wrap: wrap;
        }
      }
    }
  }
}
</style>
<style>
*:has(>.el-result) {
  display: flex;
  flex-direction: row;
  align-items: center;
  height: var(--height);
}

.el-result {
  --height: 24px;

  width: var(--height);
  height: var(--height);
  margin-left: 10px;
  padding: 0;


  & svg {
    height: var(--height);
  }
}

.el-radio-group {
  display: flex;
  flex-direction: column;
  align-items: start;
}

.options * {
  * {
    /* 自动换行 */
    text-wrap: wrap;
  }


  & button,
  & .el-radio {
    height: auto;
  }

  & .el-radio {
    margin: 5px 0;
  }

  /* 排序题的数字上下居中 */
  .el-badge__content.is-fixed {
    top: 50%;
  }

  .option {
    border: 1px solid transparent;
    border-radius: 2px;
    padding: 2px 5px;
    margin: 2px 0;

    /* 错误选项正确、正确选项样式 */
    &.option-success {
      color: green;
      border: 1px solid green;

      & span,
      & div {
        color: green;
      }
    }

    &.option-error {
      color: red;
      border: 1px solid red;

      & span,
      & div {
        color: red;
      }
    }
  }


}
</style>
