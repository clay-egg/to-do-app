<script setup lang="ts">
import { CheckIcon, PencilSquareIcon, PlusIcon, TrashIcon } from '@heroicons/vue/24/outline';
import { computed, onMounted, ref } from 'vue';

interface Todo {
    id: number;
    text: string;
    completed: boolean;
    createdAt: number;
    isEditing?: boolean;
}

const newTodo = ref('');
const todos = ref<Todo[]>([]);
const filter = ref<'all' | 'active' | 'completed'>('all');
const editedTodoText = ref('');
const activeTab = ref('all');

// Load todos from localStorage on component mount
onMounted(() => {
    const savedTodos = localStorage.getItem('todos');
    if (savedTodos) {
        todos.value = JSON.parse(savedTodos);
    }
});

// Save todos to localStorage whenever they change
const saveTodos = () => {
    localStorage.setItem('todos', JSON.stringify(todos.value));
};

const addTodo = () => {
    if (newTodo.value.trim()) {
        todos.value.unshift({
            id: Date.now(),
            text: newTodo.value.trim(),
            completed: false,
            createdAt: Date.now(),
            isEditing: false
        });
        newTodo.value = '';
        saveTodos();
    }
};

const toggleTodo = (id: number) => {
    const todo = todos.value.find(t => t.id === id);
    if (todo) {
        todo.completed = !todo.completed;
        saveTodos();
    }
};

const deleteTodo = (id: number) => {
    todos.value = todos.value.filter(t => t.id !== id);
    saveTodos();
};

const clearCompleted = () => {
    todos.value = todos.value.filter(t => !t.completed);
    saveTodos();
};

const startEditing = (todo: Todo) => {
    editedTodoText.value = todo.text;
    todo.isEditing = true;
};

const saveEdit = (todo: Todo) => {
    if (editedTodoText.value.trim()) {
        todo.text = editedTodoText.value.trim();
        todo.isEditing = false;
        saveTodos();
    }
};

const cancelEdit = (todo: Todo) => {
    todo.isEditing = false;
};

const filteredTodos = computed(() => {
    const items = [...todos.value];
    switch (filter.value) {
        case 'active':
            return items.filter(t => !t.completed);
        case 'completed':
            return items.filter(t => t.completed);
        default:
            return items;
    }
});

const activeTodosCount = computed(() => {
    return todos.value.filter(t => !t.completed).length;
});

const handleKeyDown = (e: KeyboardEvent, todo?: Todo) => {
    if (e.key === 'Enter') {
        if (todo?.isEditing) {
            saveEdit(todo);
        } else {
            addTodo();
        }
    } else if (e.key === 'Escape' && todo?.isEditing) {
        cancelEdit(todo);
    }
};

const setFilter = (type: 'all' | 'active' | 'completed') => {
    filter.value = type;
    activeTab.value = type;
};
</script>

<template>
    <div class="min-h-screen bg-gradient-to-br from-lime-50 to-green-100 p-4 sm:p-6 md:p-8">
        <div class="w-full app-container">
            <!-- Header -->
            <header class="mb-6 sm:mb-8 text-center">
                <h1 class="text-3xl sm:text-4xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-green-600 to-lime-600 mb-2">
                    My Todo List
                </h1>
                <p class="text-sm sm:text-base text-gray-600">
                    {{
                        activeTab === 'all' ? 'All tasks' :
                            activeTab === 'active' ? 'Active tasks' : 'Completed tasks'
                    }}
                </p>
            </header>

            <!-- Add Todo -->
            <div class="bg-white rounded-xl shadow-md p-4 sm:p-6 mb-6 transition-all duration-300 hover:shadow-lg">
                <div class="flex flex-col sm:flex-row gap-3">
                    <input v-model="newTodo" @keydown.enter="addTodo" type="text" placeholder="Add a new task..."
                        class="flex-1 px-4 py-3 text-base sm:text-lg rounded-lg border border-gray-200 focus:outline-none focus:ring-2 focus:ring-lime-500 focus:border-transparent transition-all" />
                    <button @click="addTodo" class="btn-primary flex items-center justify-center gap-2 shadow-md hover:shadow-lg">
                        <PlusIcon class="h-5 w-5" />
                        <span>Add</span>
                    </button>
                </div>
            </div>

            <!-- Todo List -->
            <div class="bg-white rounded-xl shadow-md overflow-hidden">
                <!-- Tabs -->
                <div class="flex border-b border-gray-100">
                    <button v-for="tab in ['all', 'active', 'completed']" :key="tab" @click="setFilter(tab as any)"
                        :class="[
                            'flex-1 py-3 sm:py-4 px-2 text-sm sm:text-base font-medium transition-colors',
                            activeTab === tab
                                ? 'text-green-600 border-b-2 border-green-500'
                                : 'text-gray-500 hover:text-gray-700 hover:bg-gray-50'
                        ]">
                        {{ tab.charAt(0).toUpperCase() + tab.slice(1) }}
                    </button>
                </div>

                <!-- Todo Items -->
                <ul class="divide-y divide-gray-100">
                    <li v-for="todo in filteredTodos" :key="todo.id" class="group hover:bg-gray-50 transition-colors">
                        <div class="px-4 sm:px-6 py-3 sm:py-4 flex items-center">
                            <!-- Checkbox -->
                            <button @click="toggleTodo(todo.id)" :class="[
                                'flex-shrink-0 h-6 w-6 sm:h-7 sm:w-7 rounded-full border-2 flex items-center justify-center transition-colors',
                                todo.completed
                                    ? 'bg-green-100 border-green-300 text-green-600'
                                    : 'border-gray-300 hover:border-lime-400'
                            ]" aria-label="Toggle completion">
                                <CheckIcon v-if="todo.completed" class="h-4 w-4 sm:h-5 sm:w-5" />
                            </button>

                            <!-- Todo Text or Edit Input -->
                            <div class="ml-3 sm:ml-4 flex-1 min-w-0">
                                <div v-if="!todo.isEditing" @dblclick="startEditing(todo)" @touchstart.passive="startEditing(todo)"
                                    :class="[
                                        'text-base sm:text-lg text-gray-800 break-words pr-2',
                                        { 'line-through text-gray-400': todo.completed }
                                    ]">
                                    {{ todo.text }}
                                </div>
                                <input v-else v-model="editedTodoText" @keydown.enter="saveEdit(todo)"
                                    @keydown.escape="cancelEdit(todo)" @blur="saveEdit(todo)" type="text"
                                    class="w-full px-2 py-1 text-base sm:text-lg border-b border-lime-400 focus:outline-none focus:border-lime-600"
                                    v-focus @click.stop />
                            </div>

                            <!-- Actions -->
                            <div
                                class="ml-2 flex items-center space-x-1 sm:space-x-2 opacity-0 group-hover:opacity-100 transition-opacity">
                                <button v-if="!todo.isEditing" @click="startEditing(todo)"
                                    class="p-1.5 sm:p-2 text-gray-400 hover:text-lime-500 rounded-full hover:bg-lime-50"
                                    title="Edit" aria-label="Edit todo">
                                    <PencilSquareIcon class="h-5 w-5" />
                                </button>
                                <button @click="deleteTodo(todo.id)"
                                    class="p-1.5 sm:p-2 text-gray-400 hover:text-red-500 rounded-full hover:bg-red-50"
                                    title="Delete" aria-label="Delete todo">
                                    <TrashIcon class="h-5 w-5" />
                                </button>
                            </div>
                        </div>
                    </li>

                    <!-- Empty State -->
                    <li v-if="filteredTodos.length === 0" class="py-12 text-center">
                        <div class="text-gray-400">
                            <div class="mx-auto w-16 h-16 mb-4 opacity-50">
                                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"
                                    stroke="currentColor">
                                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1"
                                        d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2" />
                                </svg>
                            </div>
                            <p class="text-lg font-medium">No tasks found</p>
                            <p class="text-sm">Start by adding a new task above</p>
                        </div>
                    </li>
                </ul>

                <!-- Footer -->
                <div v-if="todos.length > 0"
                    class="px-4 sm:px-6 py-3 bg-gray-50 text-sm text-gray-500 flex flex-col sm:flex-row justify-between items-center border-t border-gray-100">
                    <div class="mb-2 sm:mb-0">
                        {{ activeTodosCount }} {{ activeTodosCount === 1 ? 'item' : 'items' }} left
                    </div>
                    <div class="flex items-center space-x-4">
                        <button v-if="todos.some(t => t.completed)" @click="clearCompleted"
                            class="text-gray-500 hover:text-red-500 transition-colors flex items-center py-1 px-2 rounded hover:bg-gray-100">
                            <TrashIcon class="h-4 w-4 mr-1" />
                            <span class="text-sm">Clear completed</span>
                        </button>
                    </div>
                </div>
            </div>

        </div>
    </div>
</template>

<style scoped>
/* Component-specific styles (if needed) */
/* For example, you might want to style the tab buttons differently here */
</style>