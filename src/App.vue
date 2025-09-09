<script setup lang="ts">
import { CheckIcon, PencilSquareIcon, PlusIcon, TrashIcon } from '@heroicons/vue/24/outline';
import { computed, onMounted, ref } from 'vue';

interface Todo {
    id: number;
    title: string;
    completed: boolean;
    createdAt?: string;
    isEditing?: boolean;
}

const newTodo = ref('');
const todos = ref<Todo[]>([]);
const filter = ref<'all' | 'active' | 'completed'>('all');
const editedTodoText = ref('');
const activeTab = ref('all');
const loading = ref(false);
const error = ref('');

const API_BASE_URL = 'http://localhost:8080';

// API helper functions with better error handling
const apiRequest = async (endpoint: string, options: RequestInit = {}) => {
    try {
        console.log(`Making API request to: ${API_BASE_URL}${endpoint}`);
        
        const response = await fetch(`${API_BASE_URL}${endpoint}`, {
            headers: {
                'Content-Type': 'application/json',
                ...options.headers,
            },
            ...options,
        });

        console.log(`Response status: ${response.status}`);
        
        if (!response.ok) {
            // Try to get error message from response
            let errorMessage = `HTTP error! status: ${response.status}`;
            try {
                const errorText = await response.text();
                console.log('Error response:', errorText);
                errorMessage = errorText || errorMessage;
            } catch (e) {
                console.log('Could not parse error response');
            }
            throw new Error(errorMessage);
        }

        const contentType = response.headers.get('content-type');
        console.log('Content-Type:', contentType);
        
        if (contentType && contentType.includes('application/json')) {
            const data = await response.json();
            console.log('API Response:', data);
            return data;
        } else {
            const text = await response.text();
            console.log('Non-JSON response:', text);
            throw new Error('Expected JSON response but received: ' + text.substring(0, 100));
        }
    } catch (err) {
        console.error('API request failed:', err);
        if (err instanceof TypeError && err.message.includes('fetch')) {
            error.value = 'Cannot connect to server. Make sure the backend is running on ' + API_BASE_URL;
        } else {
            error.value = err instanceof Error ? err.message : 'An error occurred';
        }
        throw err;
    }
};

// Test connection function
const testConnection = async () => {
    try {
        loading.value = true;
        error.value = '';
        console.log('Testing connection to backend...');
        
        // Try a simple fetch first
        const response = await fetch(`${API_BASE_URL}/todos`);
        console.log('Connection test response:', response.status);
        
        if (response.status === 0) {
            throw new Error('Network error - cannot reach server');
        }
        
        return true;
    } catch (err) {
        console.error('Connection test failed:', err);
        error.value = 'Cannot connect to backend server. Please check if it\'s running on ' + API_BASE_URL;
        return false;
    } finally {
        loading.value = false;
    }
};

// Load todos from API on component mount
onMounted(async () => {
    const connected = await testConnection();
    if (connected) {
        await fetchTodos();
    }
});

const fetchTodos = async () => {
    try {
        loading.value = true;
        error.value = '';
        const data = await apiRequest('/todos');
        
        // Handle different response formats
        const todoArray = Array.isArray(data) ? data : (data.todos || []);
        
        todos.value = todoArray.map((todo: any) => ({
            id: todo.id,
            title: todo.title || todo.text || '', // Handle both title and text fields
            completed: Boolean(todo.completed),
            createdAt: todo.createdAt,
            isEditing: false
        }));
        
        console.log('Loaded todos:', todos.value);
    } catch (err) {
        console.error('Failed to fetch todos:', err);
    } finally {
        loading.value = false;
    }
};

const addTodo = async () => {
    if (!newTodo.value.trim()) return;

    try {
        loading.value = true;
        error.value = '';
        const newTodoData = await apiRequest('/todos', {
            method: 'POST',
            body: JSON.stringify({
                title: newTodo.value.trim()
            }),
        });

        todos.value.unshift({
            id: newTodoData.id,
            title: newTodoData.title,
            completed: newTodoData.completed || false,
            createdAt: newTodoData.createdAt,
            isEditing: false
        });
        newTodo.value = '';
    } catch (err) {
        console.error('Failed to add todo:', err);
    } finally {
        loading.value = false;
    }
};

const toggleTodo = async (id: number) => {
    const todo = todos.value.find(t => t.id === id);
    if (!todo) return;

    const originalCompleted = todo.completed;
    // Optimistic update
    todo.completed = !todo.completed;

    try {
        const updatedTodo = await apiRequest(`/todos/${id}`, {
            method: 'PUT',
            body: JSON.stringify({
                title: todo.title,
                completed: todo.completed
            }),
        });

        // Update with server response
        todo.completed = updatedTodo.completed;
    } catch (err) {
        // Rollback on error
        todo.completed = originalCompleted;
        console.error('Failed to toggle todo:', err);
    }
};

const deleteTodo = async (id: number) => {
    const todoIndex = todos.value.findIndex(t => t.id === id);
    if (todoIndex === -1) return;
    
    const todoBackup = todos.value[todoIndex];
    // Optimistic update
    todos.value = todos.value.filter(t => t.id !== id);

    try {
        await apiRequest(`/todos/${id}`, {
            method: 'DELETE',
        });
    } catch (err) {
        // Rollback on error
        todos.value.splice(todoIndex, 0, todoBackup);
        console.error('Failed to delete todo:', err);
    }
};

const clearCompleted = async () => {
    const completedTodos = todos.value.filter(t => t.completed);
    
    try {
        // Delete all completed todos
        await Promise.all(
            completedTodos.map(todo => 
                apiRequest(`/todos/${todo.id}`, { method: 'DELETE' })
            )
        );
        
        todos.value = todos.value.filter(t => !t.completed);
    } catch (err) {
        console.error('Failed to clear completed todos:', err);
        // Refresh the list to get current state
        await fetchTodos();
    }
};

const startEditing = (todo: Todo) => {
    editedTodoText.value = todo.title;
    todo.isEditing = true;
};

const saveEdit = async (todo: Todo) => {
    if (!editedTodoText.value.trim()) {
        cancelEdit(todo);
        return;
    }

    const originalTitle = todo.title;
    // Optimistic update
    todo.title = editedTodoText.value.trim();
    todo.isEditing = false;

    try {
        const updatedTodo = await apiRequest(`/todos/${todo.id}`, {
            method: 'PUT',
            body: JSON.stringify({
                title: todo.title,
                completed: todo.completed
            }),
        });

        todo.title = updatedTodo.title;
    } catch (err) {
        // Rollback on error
        todo.title = originalTitle;
        todo.isEditing = true;
        console.error('Failed to update todo:', err);
    }
};

const cancelEdit = (todo: Todo) => {
    todo.isEditing = false;
    editedTodoText.value = '';
};

const retryConnection = async () => {
    await fetchTodos();
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

            <!-- Error Message -->
            <div v-if="error" class="bg-red-50 border border-red-200 text-red-700 px-4 py-3 rounded-lg mb-4">
                <div class="flex justify-between items-start">
                    <div class="flex-1">
                        <h4 class="font-medium">Connection Error</h4>
                        <p class="text-sm mt-1">{{ error }}</p>
                        <div class="mt-3 flex space-x-2">
                            <button @click="retryConnection" class="text-sm bg-red-100 hover:bg-red-200 px-3 py-1 rounded">
                                Retry
                            </button>
                            <button @click="error = ''" class="text-sm text-red-600 hover:text-red-800">
                                Dismiss
                            </button>
                        </div>
                    </div>
                </div>
            </div>

            <!-- Connection Status -->
            <div v-if="loading" class="text-center mb-4">
                <div class="inline-flex items-center px-4 py-2 bg-white rounded-lg shadow">
                    <svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-lime-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                    </svg>
                    Connecting to server...
                </div>
            </div>

            <!-- Add Todo -->
            <div class="bg-white rounded-xl shadow-md p-4 sm:p-6 mb-6 transition-all duration-300 hover:shadow-lg">
                <div class="flex flex-col sm:flex-row gap-3">
                    <input 
                        v-model="newTodo" 
                        @keydown.enter="addTodo" 
                        type="text" 
                        placeholder="Add a new task..."
                        :disabled="loading"
                        class="flex-1 px-4 py-3 text-base sm:text-lg rounded-lg border border-gray-200 focus:outline-none focus:ring-2 focus:ring-lime-500 focus:border-transparent transition-all disabled:opacity-50 disabled:cursor-not-allowed" />
                    <button 
                        @click="addTodo" 
                        :disabled="loading || !newTodo.trim()"
                        class="btn-primary flex items-center justify-center gap-2 shadow-md hover:shadow-lg disabled:opacity-50 disabled:cursor-not-allowed">
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
                                <div v-if="!todo.isEditing" @dblclick="startEditing(todo)"
                                    :class="[
                                        'text-base sm:text-lg text-gray-800 break-words pr-2',
                                        { 'line-through text-gray-400': todo.completed }
                                    ]">
                                    {{ todo.title }}
                                </div>
                                <input v-else v-model="editedTodoText" @keydown.enter="saveEdit(todo)"
                                    @keydown.escape="cancelEdit(todo)" @blur="saveEdit(todo)" type="text"
                                    class="w-full px-2 py-1 text-base sm:text-lg border-b border-lime-400 focus:outline-none focus:border-lime-600"
                                    @click.stop />
                            </div>

                            <!-- Actions -->
                            <div class="ml-2 flex items-center space-x-1 sm:space-x-2 opacity-0 group-hover:opacity-100 transition-opacity">
                                <button v-if="!todo.isEditing" @click="startEditing(todo)"
                                    class="p-1.5 sm:p-2 text-gray-400 hover:text-lime-500 rounded-full hover:bg-lime-50">
                                    <PencilSquareIcon class="h-5 w-5" />
                                </button>
                                <button @click="deleteTodo(todo.id)"
                                    class="p-1.5 sm:p-2 text-gray-400 hover:text-red-500 rounded-full hover:bg-red-50">
                                    <TrashIcon class="h-5 w-5" />
                                </button>
                            </div>
                        </div>
                    </li>

                    <!-- Empty State -->
                    <li v-if="filteredTodos.length === 0 && !loading && !error" class="py-12 text-center">
                        <div class="text-gray-400">
                            <div class="mx-auto w-16 h-16 mb-4 opacity-50">
                                <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
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
.btn-primary {
    @apply bg-gradient-to-r from-lime-500 to-green-500 text-white px-6 py-3 rounded-lg font-medium hover:from-lime-600 hover:to-green-600 transition-all;
}

.app-container {
    max-width: 800px;
    margin: 0 auto;
}
</style>