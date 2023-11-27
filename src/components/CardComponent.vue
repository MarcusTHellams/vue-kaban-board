<script setup lang="ts">
import { type Priority, type Task } from '@/features/tasks';
import { ChevronDown, ChevronUp, Equal, type Icon, Plus, Minus, Edit } from 'lucide-vue-next';
import { ref, nextTick } from 'vue';

const isEditing = ref(false);
const input = ref<HTMLInputElement>();
const editable = ref<HTMLDivElement>();

const task = defineProps<Task>();
const emit = defineEmits<{
  updateTask: [task: Task];
}>();

const titleRef = ref(task.title);

const prioritySettings: Record<
  Priority,
  {
    class: string;
    component: Icon;
  }
> = {
  high: {
    class: 'text-red-500',
    component: ChevronUp,
  },
  medium: {
    class: 'text-yellow-500',
    component: Equal,
  },
  low: {
    class: 'text-blue-500',
    component: ChevronDown,
  },
};

const updatePoints = (direction: 'up' | 'down') => {
  const fib = [0, 1, 2, 3, 5, 8, 13];
  const index = fib.indexOf(task.points || 0);
  const nextIndex = direction === 'up' ? index + 1 : index - 1;
  const newPoints = fib[nextIndex];
  if (newPoints !== undefined) {
    emit('updateTask', { ...task, points: newPoints });
  }
};

const updateTitle = () => {
  emit('updateTask', { ...task, title: titleRef.value });
  isEditing.value = false;
  editable.value?.blur();
};

const enterEditMode = () => {
  isEditing.value = true;
  nextTick(() => {
    input.value?.focus();
  });
};
</script>

<template>
  <div
    :draggable="true"
    @dragstart="
      ($event: DragEvent) => {
        $event.dataTransfer?.setData('id', task.id);
      }
    "
    :data-task-id="task.id"
    class="border rounded p-2 m-2 bg-gray-50"
  >
    <div class="text-base gap-2 py-2 font-base">
      <div class="flex gap-2 items-center" v-show="!isEditing">
        <button @click="enterEditMode">
          <Edit class="text-blue-700" />
        </button>
        {{ titleRef }}
      </div>
      <input
        ref="input"
        autofocus
        @blur="isEditing = false"
        v-show="isEditing"
        v-model="titleRef"
        class="w-full"
        @keydown.enter="updateTitle"
        @keydown.esc="updateTitle"
      />
    </div>
    <div class="flex justify-between gap-4 py-2 text-gray-500 text-sm items-center">
      <div class="flex gap-2 items-center">
        <div>{{ id }}</div>
        <div>{{ priority }}</div>
        <component
          :class="prioritySettings[priority].class"
          :is="prioritySettings[priority].component"
        />
      </div>
      <div class="flex items-center gap-1">
        <button @click="updatePoints('down')">
          <Minus :size="18" />
        </button>
        {{ points || 0 }}
        <button @click="updatePoints('up')">
          <Plus :size="18" />
        </button>
      </div>
    </div>
  </div>
</template>
