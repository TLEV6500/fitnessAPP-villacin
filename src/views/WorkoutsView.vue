<script setup>
import { ref, inject, onMounted, onBeforeUnmount, nextTick } from "vue";
import { useRouter } from "vue-router";
import * as bootstrap from "bootstrap"; // Import Bootstrap JS for the Modal

const router = useRouter();
const { token, user, clearAuth } = inject("auth");

// Security Check: If they somehow get here without a token, boot them.
onMounted(() => {
    if (!token.value) {
        router.push("/");
    } else {
        fetchWorkouts();
    }
});

// State
const workouts = ref([]);
const isLoading = ref(false);
const API_BASE_URL = import.meta.env.VITE_BASE_URL ?? "http://localhost:3000";

// Modal Form State
const isEditing = ref(false);
const activeWorkoutId = ref(null);
const formData = ref({
    name: "",
    duration: "", // Assuming duration in minutes
});
let workoutModal = null; // Will hold our Bootstrap modal instance

const dropdownInstances = ref([]);
let activeDropdown = null;

// Initialize Modal on mount
onMounted(() => {
    if (document.getElementById("workoutModal")) {
        workoutModal = new bootstrap.Modal(
            document.getElementById("workoutModal"),
        );
    }
});

onBeforeUnmount(() => {
    dropdownInstances.value.forEach((instance) => {
        instance.dispose();
    });
});

// --- API HELPERS --- //

// Helper to attach the Bearer token to all requests
const authFetch = async (endpoint, options = {}) => {
    const headers = {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token.value}`,
        ...options.headers,
    };
    const response = await fetch(`${API_BASE_URL}${endpoint}`, {
        ...options,
        headers,
    });
    if (!response.ok) {
        const errorData = await response.json().catch(() => ({}));
        throw new Error(errorData.message || "API request failed");
    }
    return response.json();
};

// --- CRUD OPERATIONS --- //

// Read (Get All)
// Cleaned up fetch function
const fetchWorkouts = async () => {
    isLoading.value = true;
    try {
        const data = await authFetch("/workouts/getMyWorkouts");
        workouts.value = data.workouts || data;
    } catch (error) {
        console.debug("Failed to load workouts:", error);
    } finally {
        isLoading.value = false;
    }
};

const toggleMenu = (event) => {
    const buttonElement = event.currentTarget;
    const targetDropdown =
        bootstrap.Dropdown.getOrCreateInstance(buttonElement);
    if (activeDropdown && activeDropdown !== targetDropdown) {
        activeDropdown.hide();
    }
    targetDropdown.toggle();
    activeDropdown = targetDropdown;
};

// Open Modal for Add OR Edit
const openModal = (workout = null) => {
    if (workout) {
        isEditing.value = true;
        activeWorkoutId.value = workout._id;
        formData.value = { name: workout.name, duration: workout.duration };
    } else {
        isEditing.value = false;
        activeWorkoutId.value = null;
        formData.value = { name: "", duration: "" };
    }
    workoutModal.show();
};

// Create & Update (Submit Form)
const saveWorkout = async () => {
    try {
        if (isEditing.value) {
            await authFetch(
                `/workouts/updateWorkout/${activeWorkoutId.value}`,
                {
                    method: "PATCH",
                    body: JSON.stringify(formData.value),
                },
            );
        } else {
            await authFetch("/workouts/addWorkout", {
                method: "POST",
                body: JSON.stringify(formData.value),
            });
        }

        workoutModal.hide();
        await fetchWorkouts(); // Refresh the grid
    } catch (error) {
        alert("Error saving workout: " + error.message);
    }
};

// Update (Toggle Status)
const toggleStatus = async (id, status) => {
    try {
        await authFetch(`/workouts/completeWorkoutStatus/${id}`, {
            method: "PATCH",
            headers: {
                "Content-Type": "application/json",
            },
            body: JSON.stringify({
                status,
            }),
        });
        await fetchWorkouts(); // Refresh to show the new status
    } catch (error) {
        alert("Error updating status: " + error.message);
    }
};

// Delete
const removeWorkout = async (id) => {
    if (!confirm("Are you sure you want to shred this workout record?")) return;
    try {
        await authFetch(`/workouts/deleteWorkout/${id}`, { method: "DELETE" });
        await fetchWorkouts();
    } catch (error) {
        alert("Error deleting workout: " + error.message);
    }
};
</script>

<template>
    <div class="min-vh-100 bg-tactile pb-5">
        <nav
            class="navbar navbar-light bg-white border-bottom border-2 border-dark px-4 py-3 sticky-top shadow-sm"
        >
            <div class="container-fluid">
                <span
                    class="navbar-brand mb-0 h1 fw-bold text-uppercase tracking-widest"
                >
                    Fitness App
                </span>
                <div class="d-flex align-items-center gap-3">
                    <span class="fw-semibold text-muted d-none d-sm-block">
                        {{ user?.firstName || "Athlete" }}'s Log
                    </span>
                    <button
                        @click="clearAuth"
                        class="btn btn-outline-dark rounded-0 fw-bold px-3"
                    >
                        Eject
                    </button>
                </div>
            </div>
        </nav>

        <div class="container mt-5">
            <div
                class="d-flex justify-content-between align-items-end mb-4 border-bottom border-dark border-2 pb-3"
            >
                <div>
                    <h1 class="display-5 fw-bold mb-0">The Dashboard</h1>
                    <p class="text-muted mb-0">Your active tracking grid.</p>
                </div>
                <button
                    @click="openModal()"
                    class="btn btn-dark rounded-0 btn-lg shadow-sm fw-bold"
                >
                    + Log Workout
                </button>
            </div>

            <div v-if="isLoading" class="text-center py-5">
                <div
                    class="spinner-border text-dark"
                    style="width: 3rem; height: 3rem"
                    role="status"
                >
                    <span class="visually-hidden">Loading...</span>
                </div>
            </div>

            <div v-else-if="workouts.length === 0" class="text-center py-5">
                <div
                    class="card border-2 border-dark rounded-0 bg-white p-5 shadow-solid"
                >
                    <h3 class="fw-bold mb-3">No logs found.</h3>
                    <p class="text-muted">
                        The canvas is blank. Time to put in the work.
                    </p>
                    <button
                        @click="openModal()"
                        class="btn btn-outline-dark rounded-0 fw-bold mt-2 mx-auto"
                        style="width: fit-content"
                    >
                        Start Tracking
                    </button>
                </div>
            </div>

            <div v-else class="row row-cols-1 row-cols-md-2 row-cols-lg-3 g-4">
                <div class="col" v-for="workout in workouts" :key="workout._id">
                    <div
                        class="card h-100 border-2 border-dark rounded-0 bg-white shadow-solid position-relative"
                    >
                        <div
                            class="position-absolute top-0 start-0 translate-middle-y ms-3"
                        >
                            <span
                                v-if="workout.status === 'completed'"
                                class="badge bg-success rounded-0 border border-light px-2 py-1 text-uppercase"
                                >Completed</span
                            >
                            <span
                                v-else-if="workout.status === 'ongoing'"
                                class="badge bg-dark rounded-0 border border-light px-2 py-1 text-uppercase"
                                >Ongoing</span
                            >
                            <span
                                v-else
                                class="badge bg-warning text-dark rounded-0 border border-dark px-2 py-1 text-uppercase"
                                >Pending</span
                            >
                        </div>

                        <div class="card-body pt-4">
                            <h4 class="card-title fw-bold text-uppercase mb-3">
                                {{ workout.name }}
                            </h4>

                            <div class="d-flex align-items-center mb-2">
                                <span
                                    class="text-muted fw-semibold small me-2 text-uppercase tracking-widest"
                                    >Duration:</span
                                >
                                <span class="fs-5"
                                    >{{ workout.duration }}
                                    <small>mins</small>
                                </span>
                            </div>
                        </div>

                        <div
                            class="card-footer bg-transparent border-top border-dark border-2 d-flex justify-content-end p-2"
                        >
                            <div class="dropdown">
                                <button
                                    @click.stop="toggleMenu($event)"
                                    class="btn btn-light rounded-0 border-dark fw-bold"
                                    type="button"
                                    aria-expanded="false"
                                >
                                    •••
                                </button>

                                <ul
                                    class="dropdown-menu rounded-0 border-2 border-dark shadow"
                                    style="z-index: 1050"
                                >
                                    <li>
                                        <button
                                            @click="toggleStatus(workout._id, workout.status === 'completed' ? 'pending' : 'completed')"
                                            class="dropdown-item fw-semibold"
                                        >
                                            {{
                                                workout.status === "completed"
                                                    ? "Mark Pending"
                                                    : "Mark Completed"
                                            }}
                                        </button>
                                    </li>
                                    <li>
                                        <button
                                            @click="openModal(workout)"
                                            class="dropdown-item fw-semibold"
                                        >
                                            Edit Log
                                        </button>
                                    </li>
                                    <li>
                                        <hr
                                            class="dropdown-divider border-dark"
                                        />
                                    </li>
                                    <li>
                                        <button
                                            @click="removeWorkout(workout._id)"
                                            class="dropdown-item fw-bold text-danger"
                                        >
                                            Shred Record
                                        </button>
                                    </li>
                                </ul>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div
            class="modal fade"
            id="workoutModal"
            tabindex="-1"
            aria-labelledby="workoutModalLabel"
            aria-hidden="true"
        >
            <div class="modal-dialog modal-dialog-centered">
                <div
                    class="modal-content rounded-0 border-2 border-dark shadow-solid"
                >
                    <div
                        class="modal-header border-dark border-bottom border-2 bg-light"
                    >
                        <h5
                            class="modal-title fw-bold text-uppercase"
                            id="workoutModalLabel"
                        >
                            {{ isEditing ? "Edit Log" : "New Workout Log" }}
                        </h5>
                        <button
                            type="button"
                            class="btn-close"
                            data-bs-dismiss="modal"
                            aria-label="Close"
                        ></button>
                    </div>

                    <div class="modal-body p-4">
                        <form
                            id="workoutForm"
                            @submit.prevent="saveWorkout"
                            class="d-flex flex-column gap-3"
                        >
                            <div>
                                <label class="form-label form-label-tactile"
                                    >Workout Name / Routine</label
                                >
                                <input
                                    type="text"
                                    v-model="formData.name"
                                    class="form-control input-tactile"
                                    required
                                    placeholder="e.g., Push Day, 5K Run"
                                />
                            </div>
                            <div>
                                <label class="form-label form-label-tactile"
                                    >Duration (Minutes)</label
                                >
                                <input
                                    type="number"
                                    v-model="formData.duration"
                                    class="form-control input-tactile"
                                    required
                                    placeholder="45"
                                />
                            </div>
                        </form>
                    </div>

                    <div
                        class="modal-footer border-dark border-top border-2 bg-light"
                    >
                        <button
                            type="button"
                            class="btn btn-outline-dark rounded-0 fw-bold"
                            data-bs-dismiss="modal"
                        >
                            Cancel
                        </button>
                        <button
                            type="submit"
                            form="workoutForm"
                            class="btn btn-dark rounded-0 fw-bold"
                        >
                            {{ isEditing ? "Update Log" : "Save Log" }}
                        </button>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
/* Linen Noise Background */
.bg-tactile {
    background-color: #f4f1eb;
    background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noiseFilter'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noiseFilter)' opacity='0.05'/%3E%3C/svg%3E");
}

/* Brutalist / Physical Shadows */
.shadow-solid {
    z-index: 1;
    box-shadow: 6px 6px 0px rgba(0, 0, 0, 0.9) !important;
    transition:
        transform 0.2s ease,
        box-shadow 0.2s ease,
        z-index 0s;
}
.card.shadow-solid:hover,
.card.shadow-solid:focus-within {
    z-index: 10;
    transform: translate(-2px, -2px);
    box-shadow: 8px 8px 0px rgba(0, 0, 0, 0.9) !important;
}

.tracking-widest {
    letter-spacing: 0.1em;
}

/* Input Styling */
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
    border: 2px solid #2c2c2c;
    background-color: #faf9f6;
}
.input-tactile:focus {
    border-color: #000;
    box-shadow: none;
    background-color: #ffffff;
}

/* Dropdown override to make it sharp */
.dropdown-menu {
    padding: 0;
}
.dropdown-item {
    padding: 0.75rem 1.25rem;
}
.dropdown-item:hover {
    background-color: #f8f9fa;
}
</style>
