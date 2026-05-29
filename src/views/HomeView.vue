<script setup>
import { ref, inject } from "vue";
import { useRouter } from "vue-router";

const router = useRouter();
const { setAuth } = inject("auth");

const isLogin = ref(true);
const email = ref("");
const password = ref("");

const firstName = ref("");
const lastName = ref("");
const birthDate = ref("");

const isLoading = ref(false);
const errorMessage = ref("");

const API_BASE_URL = import.meta.env.VITE_BASE_URL ?? "http://localhost:3000";

const toggleMode = () => {
    isLogin.value = !isLogin.value;
    errorMessage.value = "";
    password.value = "";
};

const handleAuth = async () => {
    isLoading.value = true;
    errorMessage.value = "";

    const endpoint = isLogin.value ? "/users/login" : "/users/register";

    const payload = {
        email: email.value,
        password: password.value,
    };

    if (!isLogin.value) {
        payload.firstName = firstName.value;
        payload.lastName = lastName.value;
        if (birthDate.value) {
            payload.birthDate = birthDate.value;
        }
    }

    try {
        const url = `${API_BASE_URL}${endpoint}`;
        const authResponse = await fetch(url, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(payload),
        });

        const authData = await authResponse.json();
        if (!authResponse.ok)
            throw new Error(authData.message || "Authentication failed.");

        if (!isLogin.value) {
            alert("Registration successful! Please log in to continue.");
            toggleMode();
            return;
        }

        const jwt = authData.access;
        const detailsResponse = await fetch(`${API_BASE_URL}/users/details`, {
            method: "GET",
            headers: {
                Authorization: `Bearer ${jwt}`,
                "Content-Type": "application/json",
            },
        });

        const detailsData = await detailsResponse.json();
        if (!detailsResponse.ok)
            throw new Error("Failed to fetch user profile.");

        setAuth(jwt, detailsData);
        router.push("/workouts");
    } catch (error) {
        errorMessage.value = error.message;
    } finally {
        isLoading.value = false;
    }
};
</script>

<template>
    <main
        class="min-vh-100 d-flex align-items-center justify-content-center py-5 bg-tactile"
    >
        <div class="container">
            <div class="row gx-lg-5 align-items-center justify-content-center">
                <div
                    class="col-lg-4 d-none d-lg-flex justify-content-end position-relative z-1"
                >
                    <div
                        class="card bg-white p-3 pb-4 shadow-lg border-0 instax-rotate"
                        style="width: 280px"
                    >
                        <div class="ratio ratio-3x4 bg-light shadow-inner">
                            <div
                                class="d-flex align-items-center justify-content-center text-muted font-monospace tracking-widest"
                            >
                                Fitness App
                            </div>
                        </div>
                        <div
                            class="text-center mt-4 fw-bold font-monospace text-dark"
                        >
                            Daily Grind
                        </div>
                    </div>
                </div>

                <div
                    class="col-12 col-md-8 col-lg-6 col-xl-5 position-relative z-2"
                >
                    <div class="card border-0 shadow bg-white p-4 p-md-5">
                        <div class="mb-4">
                            <h1 class="display-6 fw-bold text-dark mb-1">
                                {{
                                    isLogin
                                        ? "Welcome Back"
                                        : "Start Your Journey"
                                }}
                            </h1>
                            <p class="text-muted mb-0">
                                Track your workouts. Hit your goals.
                            </p>
                        </div>

                        <form
                            @submit.prevent="handleAuth"
                            class="d-flex flex-column gap-3"
                        >
                            <div v-if="!isLogin" class="row g-3">
                                <div class="col-sm-6">
                                    <label class="form-label form-label-tactile"
                                        >First Name</label
                                    >
                                    <input
                                        type="text"
                                        v-model="firstName"
                                        class="form-control form-control-lg input-tactile"
                                        required
                                        placeholder="John"
                                    />
                                </div>
                                <div class="col-sm-6">
                                    <label class="form-label form-label-tactile"
                                        >Last Name</label
                                    >
                                    <input
                                        type="text"
                                        v-model="lastName"
                                        class="form-control form-control-lg input-tactile"
                                        required
                                        placeholder="Doe"
                                    />
                                </div>
                                <div class="col-12">
                                    <label class="form-label form-label-tactile"
                                        >Birth Date (Optional)</label
                                    >
                                    <input
                                        type="date"
                                        v-model="birthDate"
                                        class="form-control form-control-lg input-tactile"
                                    />
                                </div>
                            </div>

                            <div>
                                <label class="form-label form-label-tactile"
                                    >Email Address</label
                                >
                                <input
                                    type="email"
                                    v-model="email"
                                    class="form-control form-control-lg input-tactile"
                                    required
                                    placeholder="name@example.com"
                                />
                            </div>

                            <div>
                                <label class="form-label form-label-tactile"
                                    >Password</label
                                >
                                <input
                                    type="password"
                                    v-model="password"
                                    class="form-control form-control-lg input-tactile"
                                    required
                                    placeholder="••••••••"
                                />
                            </div>

                            <div
                                v-if="errorMessage"
                                class="text-danger fw-semibold small mt-1"
                            >
                                {{ errorMessage }}
                            </div>

                            <button
                                type="submit"
                                class="btn btn-dark btn-lg rounded-0 mt-2 fw-bold"
                                :disabled="isLoading"
                            >
                                <span
                                    v-if="isLoading"
                                    class="spinner-border spinner-border-sm me-2"
                                    role="status"
                                    aria-hidden="true"
                                ></span>
                                {{
                                    isLoading
                                        ? "Processing..."
                                        : isLogin
                                          ? "Sign In"
                                          : "Create Account"
                                }}
                            </button>
                        </form>

                        <div class="text-center mt-4 text-muted small">
                            {{
                                isLogin
                                    ? "Don't have an account?"
                                    : "Already tracking workouts?"
                            }}
                            <button
                                type="button"
                                class="btn btn-link p-0 text-dark fw-bold text-decoration-underline ms-1 align-baseline"
                                @click="toggleMode"
                            >
                                {{ isLogin ? "Sign up here" : "Log in here" }}
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </main>
</template>

<style scoped>
/* 1. Linen Noise Background */
.bg-tactile {
    background-color: #f4f1eb;
    background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noiseFilter'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noiseFilter)' opacity='0.05'/%3E%3C/svg%3E");
}

/* 2. Photo specific effects */
.instax-rotate {
    transform: rotate(-3deg);
    transition: transform 0.3s ease;
}
.instax-rotate:hover {
    transform: rotate(-1deg) scale(1.02);
}
.shadow-inner {
    box-shadow: inset 0px 2px 6px rgba(0, 0, 0, 0.08);
}
.tracking-widest {
    letter-spacing: 0.1em;
}

/* 3. Brutalist/Tactile Input Tweaks */
.form-label-tactile {
    font-size: 0.8rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    color: #444;
    margin-bottom: 0.25rem;
}
.input-tactile {
    border-radius: 0;
    border: 2px solid #e5e5e5;
    background-color: #faf9f6;
}
.input-tactile:focus {
    border-color: #2c2c2c;
    box-shadow: none;
    background-color: #ffffff;
}
</style>
