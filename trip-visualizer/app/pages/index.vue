<script setup lang="ts">
import { computed, ref, watch } from "vue";
import * as XLSX from "xlsx";
import {
  LMap,
  LTileLayer,
  LPolyline,
  LCircleMarker,
  LPopup,
} from "@vue-leaflet/vue-leaflet";

type Coordinates = {
  lat: number;
  lng: number;
};

type Trip = {
  id: number;
  date: string;
  departureTime: string;
  arrivalTime: string;
  duration: string;
  departureCoords: Coordinates | null;
  destinationCoords: Coordinates | null;
  distanceKm: number | null;
  odometerKm: number | null;
  avgConsumption: number | null;
  fuelPrice: number | null;
  cost: number | null;
  category: string;
};

const DEFAULT_CENTER: Coordinates = { lat: 3.139, lng: 101.6869 };

const trips = ref<Trip[]>([]);
const selectedDate = ref<string | null>(null);
const isLoading = ref(false);
const errorMessage = ref<string | null>(null);
const fileInput = ref<HTMLInputElement | null>(null);

const dates = computed(() =>
  Array.from(new Set(trips.value.map((trip) => trip.date))),
);

const tripCountByDate = computed(() => {
  const counts: Record<string, number> = {};
  trips.value.forEach((trip) => {
    counts[trip.date] = (counts[trip.date] ?? 0) + 1;
  });
  return counts;
});

watch(dates, (value) => {
  if (!value.length) {
    selectedDate.value = null;
    return;
  }
  if (!selectedDate.value || !value.includes(selectedDate.value)) {
    selectedDate.value = value[0];
  }
});

const filteredTrips = computed(() => {
  if (!selectedDate.value) {
    return trips.value;
  }
  return trips.value.filter((trip) => trip.date === selectedDate.value);
});

const distanceTotal = computed(() =>
  filteredTrips.value.reduce((sum, trip) => sum + (trip.distanceKm ?? 0), 0),
);

const hasCost = computed(() =>
  filteredTrips.value.some((trip) => trip.cost !== null),
);

const costTotal = computed(() =>
  filteredTrips.value.reduce((sum, trip) => sum + (trip.cost ?? 0), 0),
);

const hasConsumption = computed(() =>
  filteredTrips.value.some((trip) => trip.avgConsumption !== null),
);

const avgConsumption = computed(() => {
  const values = filteredTrips.value
    .map((trip) => trip.avgConsumption)
    .filter((value): value is number => value !== null);
  if (!values.length) {
    return null;
  }
  return values.reduce((sum, value) => sum + value, 0) / values.length;
});

const mapPoints = computed(() => {
  const points: Array<{
    key: string;
    coords: Coordinates;
    type: "departure" | "arrival";
    trip: Trip;
    title: string;
    subtitle: string;
  }> = [];
  filteredTrips.value.forEach((trip) => {
    if (trip.departureCoords) {
      points.push({
        key: `trip-${trip.id}-departure`,
        coords: trip.departureCoords,
        type: "departure",
        trip,
        title: `Departure • ${trip.departureTime}`,
        subtitle: formatCoords(trip.departureCoords),
      });
    }
    if (trip.destinationCoords) {
      points.push({
        key: `trip-${trip.id}-arrival`,
        coords: trip.destinationCoords,
        type: "arrival",
        trip,
        title: `Arrival • ${trip.arrivalTime}`,
        subtitle: formatCoords(trip.destinationCoords),
      });
    }
  });
  return points;
});

const mapSegments = computed(() => {
  const segments: Array<[number, number][]> = [];
  filteredTrips.value.forEach((trip) => {
    if (trip.departureCoords && trip.destinationCoords) {
      segments.push([
        [trip.departureCoords.lat, trip.departureCoords.lng],
        [trip.destinationCoords.lat, trip.destinationCoords.lng],
      ]);
    }
  });
  return segments;
});

const mapCenter = computed<[number, number]>(() => {
  const coords: Coordinates[] = [];
  filteredTrips.value.forEach((trip) => {
    if (trip.departureCoords) {
      coords.push(trip.departureCoords);
    }
    if (trip.destinationCoords) {
      coords.push(trip.destinationCoords);
    }
  });
  if (!coords.length) {
    return [DEFAULT_CENTER.lat, DEFAULT_CENTER.lng];
  }
  const sums = coords.reduce(
    (acc, point) => {
      acc.lat += point.lat;
      acc.lng += point.lng;
      return acc;
    },
    { lat: 0, lng: 0 },
  );
  return [sums.lat / coords.length, sums.lng / coords.length];
});

const mapZoom = computed(() => (filteredTrips.value.length > 1 ? 11 : 13));

const tableColumns = [
  { id: "departureTime", key: "departureTime", label: "Departure" },
  { id: "arrivalTime", key: "arrivalTime", label: "Arrival" },
  { id: "duration", key: "duration", label: "Duration" },
  { id: "distance", key: "distance", label: "Distance (km)" },
  { id: "odometer", key: "odometer", label: "Odometer (km)" },
  { id: "category", key: "category", label: "Category" },
];

const tableRows = computed(() =>
  filteredTrips.value.map((trip) => ({
    departureTime: trip.departureTime || "–",
    arrivalTime: trip.arrivalTime || "–",
    duration: trip.duration || "–",
    distance: trip.distanceKm !== null ? trip.distanceKm.toFixed(1) : "–",
    odometer:
      trip.odometerKm !== null
        ? Math.round(trip.odometerKm).toLocaleString()
        : "–",
    category: trip.category || "–",
  })),
);

function triggerFileInput() {
  errorMessage.value = null;
  if (fileInput.value) {
    fileInput.value.click();
  }
}

async function handleFileChange(event: Event) {
  const input = event.target as HTMLInputElement;
  const file = input.files ? input.files[0] : undefined;
  if (!file) {
    return;
  }
  isLoading.value = true;
  errorMessage.value = null;
  try {
    const buffer = await file.arrayBuffer();
    const workbook = XLSX.read(buffer, { type: "array" });
    const firstSheetName = workbook.SheetNames[0];
    const sheet = workbook.Sheets[firstSheetName];
    const rows = XLSX.utils.sheet_to_json(sheet, {
      header: 1,
      defval: "",
      blankrows: false,
    }) as unknown[];
    if (!rows.length) {
      throw new Error("The uploaded file does not contain any rows.");
    }
    const headers = normalizeRow(rows[0]);
    const dataRows = (rows as unknown[]).slice(1).map((row) => {
      const cells = normalizeRow(row);
      if (cells.length < headers.length) {
        return [...cells, ...Array(headers.length - cells.length).fill("")];
      }
      if (cells.length > headers.length) {
        return cells.slice(0, headers.length);
      }
      return cells;
    });
    const parsedTrips = dataRows
      .map((cells, index) => buildTripFromRow(headers, cells, index))
      .filter((trip): trip is Trip => trip !== null);
    if (!parsedTrips.length) {
      throw new Error(
        "No valid trips were detected. Ensure the spreadsheet is semicolon-delimited and contains the expected headers.",
      );
    }
    trips.value = parsedTrips;
  } catch (err) {
    errorMessage.value =
      err instanceof Error
        ? err.message
        : "Unable to process the uploaded file.";
    trips.value = [];
  } finally {
    isLoading.value = false;
    if (fileInput.value) {
      fileInput.value.value = "";
    }
  }
}

function normalizeRow(row: unknown): string[] {
  if (Array.isArray(row)) {
    if (
      row.length === 1 &&
      typeof row[0] === "string" &&
      row[0].includes(";")
    ) {
      return splitSemicolonRow(row[0]);
    }
    return row.map((cell) =>
      typeof cell === "string" ? sanitize(cell) : sanitize(String(cell)),
    );
  }
  if (typeof row === "string") {
    return splitSemicolonRow(row);
  }
  return [];
}

function splitSemicolonRow(value: string): string[] {
  const cells: string[] = [];
  let current = "";

  for (const char of value) {
    if (char === ";") {
      cells.push(current);
      current = "";
    } else {
      current += char;
    }
  }

  cells.push(current);
  return cells.map((cell) => sanitize(cell));
}

function sanitize(value: string): string {
  return value.replace(/[\u00a0\u202f]/g, " ").trim();
}

function normalizeHeader(value: string): string {
  return sanitize(value).toLowerCase().replace(/\s+/g, " ");
}

function buildTripFromRow(
  headers: string[],
  cells: string[],
  index: number,
): Trip | null {
  if (!cells.length || cells.every((cell) => !cell.trim())) {
    return null;
  }
  const data: Record<string, string> = {};
  headers.forEach((header, headerIndex) => {
    data[normalizeHeader(header)] = sanitize(cells[headerIndex] ?? "");
  });
  const fetchValue = (...keys: string[]) => {
    for (const key of keys) {
      const normalized = normalizeHeader(key);
      if (data[normalized]) {
        return data[normalized];
      }
    }
    return "";
  };
  const departureCoords = parseCoordinates(
    fetchValue("address of departure", "departure address", "origin"),
  );
  const destinationCoords = parseCoordinates(
    fetchValue("destination address", "destination", "destination coordinate"),
  );
  return {
    id: index,
    date: fetchValue("date"),
    departureTime: fetchValue("time of departure"),
    arrivalTime: fetchValue("time of arrival"),
    duration: fetchValue("time (hr:min)"),
    departureCoords,
    destinationCoords,
    distanceKm: parseNumber(fetchValue("distance (km)")),
    odometerKm: parseNumber(fetchValue("mileage on odometer (km)")),
    avgConsumption: parseNumber(
      fetchValue("avg. consumption (l/100km)", "avg. consumption  (l/100km)"),
    ),
    fuelPrice: parseNumber(fetchValue("price of fuel (sgd/l)")),
    cost: parseNumber(fetchValue("cost (sgd)")),
    category: fetchValue("category"),
  };
}

function parseNumber(value: string): number | null {
  if (!value) {
    return null;
  }
  const cleaned = value.replace(/[^0-9,.-]/g, "").replace(",", ".");
  const numeric = Number(cleaned);
  return Number.isFinite(numeric) ? numeric : null;
}

function parseCoordinates(value: string): Coordinates | null {
  if (!value) {
    return null;
  }
  const parts = value.split(",").map((part) => part.trim());
  if (parts.length !== 2) {
    return null;
  }
  const lat = Number(parts[0]);
  const lng = Number(parts[1]);
  if (!Number.isFinite(lat) || !Number.isFinite(lng)) {
    return null;
  }
  return { lat, lng };
}

function formatCoords(coords: Coordinates): string {
  return `${coords.lat.toFixed(5)}, ${coords.lng.toFixed(5)}`;
}
</script>

<template>
  <UContainer class="space-y-8 py-10">
    <UPageHero
      title="Trip Visualizer"
      description="Upload a semicolon-delimited XLSX file to review daily trips and explore them on an interactive map."
      :links="[]"
    />

    <UCard>
      <template #header>
        <div class="flex items-center justify-between">
          <h2 class="text-lg font-semibold">Upload trip log</h2>
          <span class="text-sm text-muted"> Accepted format: .xlsx </span>
        </div>
      </template>

      <div
        class="flex flex-col gap-4 md:flex-row md:items-center md:justify-between"
      >
        <p class="text-sm text-muted">
          Select an XLSX file exported with semicolon-separated values. The
          first sheet will be processed automatically. This file supports the
          MyPeugeot trip log format, but any spreadsheet with the expected
          headers can be used.
        </p>

        <div class="flex items-center gap-3">
          <UButton
            color="primary"
            icon="i-lucide-upload"
            :loading="isLoading"
            @click="triggerFileInput"
          >
            Choose file
          </UButton>
          <span v-if="isLoading" class="text-sm text-muted"> Processing… </span>
        </div>

        <input
          ref="fileInput"
          type="file"
          class="hidden"
          accept=".xlsx,.xls"
          @change="handleFileChange"
        />
      </div>

      <ul class="mt-4 space-y-1 text-sm text-muted">
        <li>
          Expected headers: Date, Time of Departure, Time of Arrival, Time
          (hr:min), Address of Departure, Destination Address, Distance (km),
          Mileage on odometer (km), Avg. consumption (l/100km), Price of fuel
          (MYR/l), Cost (MYR), Category.
        </li>
      </ul>
    </UCard>

    <UAlert
      v-if="errorMessage"
      color="error"
      icon="i-lucide-alert-triangle"
      variant="soft"
    >
      {{ errorMessage }}
    </UAlert>

    <div v-if="trips.length" class="grid gap-6 lg:grid-cols-[320px_1fr]">
      <div class="space-y-4">
        <UCard>
          <template #header>
            <h3 class="text-base font-semibold">Dates</h3>
          </template>

          <div class="flex flex-col gap-2">
            <UButton
              v-for="date in dates"
              :key="date"
              :variant="date === selectedDate ? 'solid' : 'soft'"
              :color="date === selectedDate ? 'primary' : 'neutral'"
              class="justify-between"
              @click="selectedDate = date"
            >
              <span>
                {{ date }}
              </span>
              <span class="text-sm text-muted">
                ({{ tripCountByDate[date] ?? 0 }} trips)
              </span>
            </UButton>
          </div>
        </UCard>

        <UCard v-if="filteredTrips.length">
          <template #header>
            <h3 class="text-base font-semibold">Summary</h3>
          </template>

          <ul class="space-y-3">
            <li class="flex items-center justify-between">
              <span class="text-sm text-muted"> Total distance </span>
              <span class="font-medium">
                {{ distanceTotal.toFixed(1) }} km
              </span>
            </li>
            <li class="flex items-center justify-between">
              <span class="text-sm text-muted"> Fuel cost </span>
              <span class="font-medium">
                {{ hasCost ? `${costTotal.toFixed(2)} SGD` : "–" }}
              </span>
            </li>
            <li class="flex items-center justify-between">
              <span class="text-sm text-muted"> Avg. consumption </span>
              <span class="font-medium">
                {{
                  hasConsumption && avgConsumption !== null
                    ? `${avgConsumption.toFixed(2)} l/100km`
                    : "–"
                }}
              </span>
            </li>
          </ul>
        </UCard>
      </div>

      <div class="space-y-4">
        <UCard>
          <template #header>
            <div class="flex items-center justify-between">
              <h3 class="text-base font-semibold">Map</h3>
              <span v-if="selectedDate" class="text-sm text-muted">
                Showing {{ filteredTrips.length }} trip(s)
              </span>
            </div>
          </template>

          <div class="overflow-hidden rounded-md">
            <ClientOnly>
              <template #fallback>
                <div
                  class="flex h-[480px] items-center justify-center bg-muted/40 text-sm text-muted"
                >
                  Loading map…
                </div>
              </template>

              <LMap
                v-if="filteredTrips.length"
                :center="mapCenter"
                :zoom="mapZoom"
                :use-global-leaflet="false"
                style="height: 480px"
              >
                <LTileLayer
                  url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
                />

                <LPolyline
                  v-for="(segment, index) in mapSegments"
                  :key="`segment-${index}`"
                  :lat-lngs="segment"
                  color="#2563eb"
                  :weight="4"
                  :opacity="0.6"
                />

                <LCircleMarker
                  v-for="point in mapPoints"
                  :key="point.key"
                  :lat-lng="[point.coords.lat, point.coords.lng]"
                  :radius="point.type === 'departure' ? 10 : 8"
                  :color="point.type === 'departure' ? '#22c55e' : '#ef4444'"
                  :fill-color="
                    point.type === 'departure' ? '#22c55e' : '#ef4444'
                  "
                  :fill-opacity="0.75"
                >
                  <LPopup>
                    <div class="space-y-1 text-sm">
                      <p class="font-semibold">
                        {{ point.title }}
                      </p>
                      <p class="text-muted">
                        {{ point.subtitle }}
                      </p>
                      <p>
                        Distance:
                        {{
                          point.trip.distanceKm !== null
                            ? `${point.trip.distanceKm.toFixed(1)} km`
                            : "–"
                        }}
                      </p>
                      <p>Category: {{ point.trip.category || "–" }}</p>
                    </div>
                  </LPopup>
                </LCircleMarker>
              </LMap>

              <div
                v-else
                class="flex h-[480px] items-center justify-center bg-muted/40 text-sm text-muted"
              >
                Select a date with trips to view the map.
              </div>
            </ClientOnly>
          </div>
        </UCard>

        <UCard v-if="filteredTrips.length">
          <template #header>
            <h3 class="text-base font-semibold">
              Trips for {{ selectedDate || "all dates" }}
            </h3>
          </template>

          <UTable :columns="tableColumns" :rows="tableRows" />
        </UCard>

        <div
          v-else
          class="rounded-md border border-dashed border-muted p-8 text-center text-sm text-muted"
        >
          Select a date to view trip details.
        </div>
      </div>
    </div>

    <div
      v-else
      class="rounded-md border border-dashed border-muted p-12 text-center text-sm text-muted"
    >
      Upload a trip log to visualize your journeys.
    </div>
  </UContainer>
</template>
