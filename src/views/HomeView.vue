<script setup lang="ts">
import Card from '@/components/CardComponent.vue';
import { statuses, type Task, type Status } from '@/features/tasks';
import { ref, computed } from 'vue';
import { useQuery, useMutation, useQueryClient } from '@tanstack/vue-query';
import axios from 'axios';

const queryClient = useQueryClient();

const api = axios.create({
  baseURL: 'http://localhost:3000/tasks',
});

const { data: tasks } = useQuery<Task[], Error>({
  queryKey: ['tasks'],
  async queryFn() {
    return api.get('').then(({ data }) => data);
  },
  placeholderData: () => {
    return [];
  },
});

const { mutate } = useMutation<Task, Error, Task>({
  mutationKey: ['updateTask'],
  async mutationFn(task) {
    return api.put(`/${task.id}`, task).then(({ data }) => data);
  },
  async onSettled() {
    return await queryClient.invalidateQueries({ queryKey: ['tasks'] });
  },
  onMutate: async (updatedTask) => {
    // Cancel any outgoing refetches
    // (so they don't overwrite our optimistic update)
    await queryClient.cancelQueries({ queryKey: ['tasks'] });

    const previousTasks = queryClient.getQueryData(['tasks']) as Task[];

    // Optimistically update to the new value
    queryClient.setQueryData(['tasks'], (old: Task[] | undefined) => {
      if (old) {
        return old.map((task) => {
          if (task.id === updatedTask.id) {
            return updatedTask;
          }
          return task;
        });
      }
      return undefined;
    });

    return { previousTasks };
  },
  // If the mutation fails,
  // use the context returned from onMutate to roll back
  onError: (_, __, context) => {
    const _context = context as { previousTasks: Task[] };
    queryClient.setQueryData(['tasks'], _context.previousTasks);
  },
});

const columns = computed(() => {
  if (tasks.value) {
    return statuses.map((status) => {
      const tasksInColumn = tasks.value.filter((task) => task.status === status);
      return {
        title: status,
        tasksInColumn,
      };
    });
  }
  return statuses.map((status) => {
    return {
      title: status,
      tasksInColumn: [],
    };
  });
});

const updateTask = (task: Task) => {
  mutate(task);
};

const handleDrop = (event: DragEvent, title: Status) => {
  const id = event.dataTransfer?.getData('id');
  if (!id || !tasks.value) return;
  const task = tasks.value.filter((task) => task.id === id)[0];
  updateTask({ ...task, status: title });
  currentlyHoveringOver.value = null;
};

const currentlyHoveringOver = ref<Status | null>(null);

const handleDragEnter = (status: Status) => {
  currentlyHoveringOver.value = status;
};
</script>

<template>
  <div class="flex divide-x">
    <div
      @drop="handleDrop($event, column.title)"
      @dragover.prevent
      @dragenter="handleDragEnter(column.title)"
      class="grow"
      v-for="column in columns"
      :key="column.title"
    >
      <div class="flex justify-between text-3xl capitalize font-bold text-gray-500 p-2">
        <h2>
          {{ column.title }}
        </h2>
        <div>
          {{
            column.tasksInColumn.reduce((total, task) => {
              return total + (task?.points || 0);
            }, 0)
          }}
        </div>
      </div>
      <div :class="currentlyHoveringOver === column.title && 'bg-gray-200'" class="h-full">
        <Card
          @update-task="updateTask"
          v-for="task in column.tasksInColumn"
          :key="task.id"
          :="task"
        />
      </div>
    </div>
  </div>
</template>
