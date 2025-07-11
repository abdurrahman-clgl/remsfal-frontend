<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { garageService, type GarageUnit } from '@/services/GarageService';
import { useToast } from 'primevue/usetoast';
import { handleCancel, showSavingErrorToast, showValidationErrorToast } from '@/helper/viewHelper';

const props = defineProps<{
  projectId: string;
  unitId: string;
}>();

const toast = useToast();
const router = useRouter();

// Formfelder
const title = ref('');
const description = ref('');
const location = ref('');
const usableSpace = ref<number | null>(null);

// Originalwerte zur Vergleichserkennung
const originalValues = ref({
  title: '',
  description: '',
  location: '',
  usableSpace: null as number | null,
});

const hasChanges = computed(() => {
  return (
    title.value !== originalValues.value.title ||
    description.value !== originalValues.value.description ||
    location.value !== originalValues.value.location ||
    usableSpace.value !== originalValues.value.usableSpace
  );
});

const validationErrors = computed(() => {
  const errors: string[] = [];
  if (!title.value || title.value.length < 3) {
    errors.push('Der Titel muss mindestens 3 Zeichen lang sein.');
  }
  if (!location.value) {
    errors.push('Standort ist erforderlich.');
  }
  if (usableSpace.value === null) {
    errors.push('Nutzfläche ist erforderlich.');
  } else if (usableSpace.value < 0) {
    errors.push('Nutzfläche darf nicht negativ sein.');
  }
  if (description.value && description.value.length > 500) {
    errors.push('Beschreibung darf maximal 500 Zeichen lang sein.');
  }
  return errors;
});

const isValid = computed(() => validationErrors.value.length === 0);

const fetchGarageDetails = async () => {
  if (!props.projectId) {
    console.error('Keine projectId');
    return;
  }
  if (!props.unitId) {
    console.error('Keine unitId');
    return;
  }
  try {
    const data = await garageService.getGarage(props.projectId, props.unitId);
    console.log('Garage-Daten vom Backend:', data); // Hier siehst du die Antwort
    title.value = data.title || '';
    description.value = data.description || '';
    location.value = data.location || '';
    usableSpace.value = data.usableSpace ?? null;

    originalValues.value = {
      title: title.value,
      description: description.value,
      location: location.value,
      usableSpace: usableSpace.value,
    };
  } catch (error) {
    console.error('Fehler beim Laden der Garage:', error);
    toast.add({
      severity: 'error',
      summary: 'Ladefehler',
      detail: 'Garage konnte nicht geladen werden.',
      life: 6000,
    });
  }
};
onMounted(() => {
  if (props.unitId) {
    fetchGarageDetails();
  } else {
    console.warn('unitId fehlt – keine Daten können geladen werden.');
    toast.add({
      severity: 'warn',
      summary: 'Ungültige ID',
      detail: 'Garage konnte nicht geladen werden, da keine ID übergeben wurde.',
      life: 6000,
    });
  }
});

const save = async () => {
  if (!isValid.value) {
    showValidationErrorToast(toast, validationErrors.value);
    return;
  }

  const payload: GarageUnit = {
    title: title.value,
    description: description.value,
    location: location.value,
    usableSpace: usableSpace.value!,
  };

  try {
    await garageService.updateGarage(props.projectId, props.unitId, payload);
    toast.add({
      severity: 'success',
      summary: 'Erfolg',
      detail: 'Garage erfolgreich gespeichert.',
      life: 6000,
    });
    router.push(`/project/${props.projectId}/garage/${props.unitId}`);
  } catch (err) {
    console.error('Fehler beim Speichern:', err);
    showSavingErrorToast(toast, 'Garage konnte nicht gespeichert werden.');
  }
};

const cancel = () => handleCancel(hasChanges, router, props.projectId);
</script>

<template>
  <div class="p-6 w-full">
    <div class="bg-white rounded-lg shadow-md p-10 max-w-screen-2xl mx-auto">
      <h2 class="text-2xl font-semibold mb-6">Bearbeite Garage mit ID: {{ unitId }}</h2>
      <form @submit.prevent="save">
        <div class="grid grid-cols-1 md:grid-cols-2 gap-x-8 gap-y-4">
          <!-- Titel -->
          <div class="col-span-2">
            <label for="title" class="block text-gray-700 mb-1">Titel</label>
            <input id="title" v-model="title" type="text" class="form-input w-full" />
          </div>

          <!-- Beschreibung -->
          <div class="col-span-2">
            <label for="description" class="block text-gray-700 mb-1">Beschreibung</label>
            <textarea id="description" v-model="description" rows="3" class="form-textarea w-full"></textarea>
          </div>

          <!-- Standort -->
          <div class="col-span-2">
            <label for="location" class="block text-gray-700 mb-1">Standort</label>
            <input id="location" v-model="location" type="text" class="form-input w-full" />
          </div>

          <!-- Nutzfläche -->
          <div>
            <label for="usableSpace" class="block text-gray-700 mb-1">Nutzfläche (m²)</label>
            <input id="usableSpace" v-model.number="usableSpace" type="number" class="form-input w-full" />
          </div>
        </div>
        <!-- Validierungsfehler -->
        <div v-if="validationErrors.length" class="text-red-600 mt-4">
          <ul>
            <li v-for="(error, i) in validationErrors" :key="i">{{ error }}</li>
          </ul>
        </div>

        <!-- Buttons -->
        <div class="mt-6 flex justify-end space-x-4">
          <button
            type="submit"
            :disabled="!hasChanges"
            class="bg-green-600 text-white px-4 py-2 rounded hover:bg-green-700 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            Speichern
          </button>

          <button
            type="button"
            class="bg-gray-400 text-white px-4 py-2 rounded hover:bg-gray-500"
            @click="cancel"
          >
            Abbrechen
          </button>
        </div>
      </form>
    </div>
  </div>
</template>

<style scoped>
.form-input,
.form-textarea {
  background-color: #f9fafb;
  border-radius: 0.5rem;
  padding: 0.5rem;
  border: 1px solid #d1d5db;
}
</style>
