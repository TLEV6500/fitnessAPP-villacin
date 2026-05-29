<script setup>
import { ref, provide } from "vue";
import { RouterView, useRouter } from "vue-router";

const router = useRouter();

// 1. Initialize state (Check for existing token on load)
const token = ref(localStorage.getItem("fitness_token") || null);
const user = ref(null);

// 2. State Mutators
const setAuth = (newToken, userData) => {
    token.value = newToken;
    user.value = userData;
    localStorage.setItem("fitness_token", newToken);
};

const clearAuth = () => {
    token.value = null;
    user.value = null;
    localStorage.removeItem("fitness_token");
    router.push("/"); // Kick them back to the tactile canvas
};

// 3. Provide it to the entire application
// Using a string key 'auth' for simplicity, though Symbols are safer in massive apps
provide("auth", {
    token,
    user,
    setAuth,
    clearAuth,
});
</script>

<template>
    <RouterView />
</template>

<style>
/* Global reset and baseline typography */
* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    font-family:
        "Inter",
        -apple-system,
        BlinkMacSystemFont,
        "Segoe UI",
        Roboto,
        Helvetica,
        Arial,
        sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
}
</style>
