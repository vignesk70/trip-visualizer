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
const MALAYSIA_TIMEZONE = "Asia/Kuala_Lumpur";
const MALAYSIA_LOCALE = "en-MY";

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

function convertToMalaysia(
  dateStr: string,
  timeStr: string,
): {
  date: string;
  time: string;
} {
  const sanitizedDate = sanitize(dateStr);
  const sanitizedTime = sanitize(timeStr);
  const fallback = {
    date: sanitizedDate || dateStr,
    time: sanitizedTime || timeStr,
  };

  const [dayStr, monthStr, yearStr] = sanitizedDate.split("/");
  const day = Number(dayStr);
  const month = Number(monthStr);
  const year = Number(yearStr);
  if (!day || !month || !year) {
    return fallback;
  }

  const normalizedTime = sanitizedTime.replace(/\s+/g, " ").toUpperCase();
  const timeMatch = normalizedTime.match(/^(\d{1,2}):(\d{2})\s*(AM|PM)$/);
  if (!timeMatch) {
    return fallback;
  }

  let hours = Number(timeMatch[1]);
  const minutes = Number(timeMatch[2]);
  const period = timeMatch[3];
  if (!Number.isFinite(hours) || !Number.isFinite(minutes)) {
    return fallback;
  }

  hours %= 12;
  if (period === "PM") {
    hours += 12;
  }

  const utcDate = new Date(Date.UTC(year, month - 1, day, hours, minutes));
  if (Number.isNaN(utcDate.getTime())) {
    return fallback;
  }

  const formatter = new Intl.DateTimeFormat(MALAYSIA_LOCALE, {
    year: "numeric",
    month: "2-digit",
    day: "2-digit",
    hour: "numeric",
    minute: "2-digit",
    hour12: true,
    timeZone: MALAYSIA_TIMEZONE,
  });

  const parts = formatter.formatToParts(utcDate);
  const partMap = parts.reduce<Record<string, string>>((acc, part) => {
    if (part.type !== "literal") {
      acc[part.type] = part.value;
    }
    return acc;
  }, {});

  const localDate = [
    partMap.day ?? dayStr.padStart(2, "0"),
    partMap.month ?? monthStr.padStart(2, "0"),
    partMap.year ?? yearStr,
  ].join("/");

  const periodLabel = partMap.dayPeriod ? partMap.dayPeriod.toUpperCase() : "";
  const localTime = `${partMap.hour ?? String(hours || 12)}:${partMap.minute ?? minutes.toString().padStart(2, "0")}${periodLabel ? ` ${periodLabel}` : ""}`;

  return {
    date: localDate,
    time: localTime,
  };
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
  const rawDate = fetchValue("date");
  const rawDepartureTime = fetchValue("time of departure");
  const rawArrivalTime = fetchValue("time of arrival");

  const departureLocal = convertToMalaysia(rawDate, rawDepartureTime);
  const arrivalLocal = convertToMalaysia(rawDate, rawArrivalTime);

  const departureCoords = parseCoordinates(
    fetchValue("address of departure", "departure address", "origin"),
  );
  const destinationCoords = parseCoordinates(
    fetchValue("destination address", "destination", "destination coordinate"),
  );

  return {
    id: index,
    date: departureLocal.date,
    departureTime: departureLocal.time,
    arrivalTime: arrivalLocal.time,
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
  <div
    class="min-h-screen bg-gradient-to-b from-blue-950 via-slate-950 to-slate-900"
  >
    <UContainer class="py-12 space-y-10">
      <section
        class="grid gap-8 lg:grid-cols-[minmax(0,1fr)_380px] items-start"
      >
        <div class="space-y-6 text-white">
          <div class="flex flex-wrap items-center gap-3">
            <UBadge
              color="primary"
              variant="subtle"
              class="uppercase tracking-wide"
            >
              Modern
            </UBadge>
            <span class="text-sm text-blue-200"
              >Insights for every journey</span
            >
          </div>
          <h1 class="text-4xl font-semibold leading-tight sm:text-5xl">
            Visualize every kilometer of your trip history
          </h1>
          <p class="text-lg text-blue-100 sm:max-w-2xl">
            Upload your MyPeugeot export to plot routes, uncover driving
            patterns, and keep tabs on fuel efficiency with a sleek, interactive
            dashboard.
          </p>
          <div class="flex flex-wrap gap-3">
            <UButton
              color="primary"
              size="lg"
              icon="i-lucide-upload"
              @click="triggerFileInput"
            >
              Upload log
            </UButton>
            <UButton
              variant="ghost"
              color="primary"
              size="lg"
              @click="triggerFileInput"
            >
              Choose different file
            </UButton>
          </div>
          <div class="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
            <div class="rounded-2xl bg-white/10 backdrop-blur p-4 shadow-lg">
              <p class="text-sm text-blue-200">Trips analysed</p>
              <p class="text-2xl font-semibold">{{ trips.length }}</p>
            </div>
            <div class="rounded-2xl bg-white/10 backdrop-blur p-4 shadow-lg">
              <p class="text-sm text-blue-200">Distance logged</p>
              <p class="text-2xl font-semibold">
                {{ distanceTotal.toFixed(1) }} km
              </p>
            </div>
            <div class="rounded-2xl bg-white/10 backdrop-blur p-4 shadow-lg">
              <p class="text-sm text-blue-200">Unique days</p>
              <p class="text-2xl font-semibold">{{ dates.length }}</p>
            </div>
          </div>
        </div>

        <UCard
          class="border border-white/10 bg-gradient-to-br from-blue-900/60 to-slate-900/40 backdrop-blur"
        >
          <template #header>
            <div>
              <h2 class="text-xl font-semibold text-white">
                Upload a trip log
              </h2>
              <p class="text-sm text-blue-200">
                Drop in your semicolon-delimited XLSX export to unlock insights.
              </p>
            </div>
          </template>

          <div class="space-y-4 text-sm text-blue-100">
            <p>
              Processing happens entirely in your browser. No trip data ever
              leaves your device.
            </p>
            <ul class="space-y-2 leading-relaxed">
              <li class="flex items-start gap-2">
                <UIcon
                  name="i-lucide-check"
                  class="mt-0.5 h-4 w-4 text-primary-400"
                />
                <span
                  >Supports MyPeugeot XLSX exports with semicolon
                  separation.</span
                >
              </li>
              <li class="flex items-start gap-2">
                <UIcon
                  name="i-lucide-check"
                  class="mt-0.5 h-4 w-4 text-primary-400"
                />
                <span
                  >Automatically plots departure and destination coordinates on
                  the map.</span
                >
              </li>
              <li class="flex items-start gap-2">
                <UIcon
                  name="i-lucide-check"
                  class="mt-0.5 h-4 w-4 text-primary-400"
                />
                <span
                  >Summaries refresh instantly for each day you explore.</span
                >
              </li>
            </ul>

            <UButton
              block
              size="lg"
              color="primary"
              icon="i-lucide-folder-up"
              :loading="isLoading"
              @click="triggerFileInput"
            >
              Browse files
            </UButton>

            <input
              ref="fileInput"
              type="file"
              class="hidden"
              accept=".xlsx,.xls"
              @change="handleFileChange"
            />

            <p class="text-xs text-blue-300">
              Expected headers: Date, Time of Departure, Time of Arrival, Time
              (hr:min), Address of Departure, Destination Address, Distance
              (km), Mileage on odometer (km), Avg. consumption (l/100km), Price
              of fuel (SGD/l), Cost (SGD), Category.
            </p>
          </div>
        </UCard>
      </section>

      <UAlert
        v-if="errorMessage"
        icon="i-lucide-alert-triangle"
        color="error"
        variant="soft"
        class="border border-red-500/30 bg-red-500/10 text-red-100"
      >
        {{ errorMessage }}
      </UAlert>

      <section
        v-if="trips.length"
        class="grid gap-6 xl:grid-cols-[320px_minmax(0,1fr)]"
      >
        <UCard class="border border-white/5 bg-white/5 backdrop-blur">
          <template #header>
            <h3 class="text-base font-semibold text-white">Travel dates</h3>
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
              <span>{{ date }}</span>
              <span class="text-sm text-blue-200">
                ({{ tripCountByDate[date] ?? 0 }} trips)
              </span>
            </UButton>
          </div>
        </UCard>

        <div class="space-y-6">
          <UCard class="border border-white/5 bg-white/5 backdrop-blur">
            <template #header>
              <div class="flex flex-wrap items-center justify-between gap-3">
                <h3 class="text-base font-semibold text-white">
                  Daily overview
                </h3>
                <UBadge color="primary" variant="soft">
                  {{ selectedDate || "All dates" }}
                </UBadge>
              </div>
            </template>

            <div class="grid gap-4 sm:grid-cols-3">
              <div class="rounded-xl bg-blue-500/10 p-4">
                <p class="text-sm text-blue-200">Total distance</p>
                <p class="text-2xl font-semibold text-white">
                  {{ distanceTotal.toFixed(1) }} km
                </p>
              </div>
              <div class="rounded-xl bg-blue-500/10 p-4">
                <p class="text-sm text-blue-200">Fuel cost</p>
                <p class="text-2xl font-semibold text-white">
                  {{ hasCost ? `${costTotal.toFixed(2)} SGD` : "–" }}
                </p>
              </div>
              <div class="rounded-xl bg-blue-500/10 p-4">
                <p class="text-sm text-blue-200">Avg. consumption</p>
                <p class="text-2xl font-semibold text-white">
                  {{
                    hasConsumption && avgConsumption !== null
                      ? `${avgConsumption.toFixed(2)} l/100km`
                      : "–"
                  }}
                </p>
              </div>
            </div>
          </UCard>

          <UCard class="border border-white/5 bg-white/5 backdrop-blur">
            <template #header>
              <div class="flex flex-wrap items-center justify-between gap-3">
                <h3 class="text-base font-semibold text-white">Map</h3>
                <span class="text-sm text-blue-200">
                  Showing {{ filteredTrips.length }} trip(s)
                </span>
              </div>
            </template>

            <div class="overflow-hidden rounded-xl border border-white/10">
              <ClientOnly>
                <template #fallback>
                  <div
                    class="flex h-[460px] items-center justify-center bg-slate-800/60 text-sm text-blue-200"
                  >
                    Loading map…
                  </div>
                </template>

                <LMap
                  v-if="filteredTrips.length"
                  :center="mapCenter"
                  :zoom="mapZoom"
                  :use-global-leaflet="false"
                  style="height: 460px"
                >
                  <LTileLayer
                    url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
                  />

                  <LPolyline
                    v-for="(segment, index) in mapSegments"
                    :key="`segment-${index}`"
                    :lat-lngs="segment"
                    color="#38bdf8"
                    :weight="4"
                    :opacity="0.7"
                  />

                  <LCircleMarker
                    v-for="point in mapPoints"
                    :key="point.key"
                    :lat-lng="[point.coords.lat, point.coords.lng]"
                    :radius="point.type === 'departure' ? 10 : 8"
                    :color="point.type === 'departure' ? '#22d3ee' : '#fb7185'"
                    :fill-color="
                      point.type === 'departure' ? '#22d3ee' : '#fb7185'
                    "
                    :fill-opacity="0.75"
                  >
                    <LPopup>
                      <div class="space-y-1 text-sm">
                        <p class="font-semibold text-slate-800">
                          {{ point.title }}
                        </p>
                        <p class="text-slate-500">{{ point.subtitle }}</p>
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
                  class="flex h-[460px] items-center justify-center bg-slate-800/60 text-sm text-blue-200"
                >
                  Select a date with trips to view the map.
                </div>
              </ClientOnly>
            </div>
          </UCard>

          <UCard class="border border-white/5 bg-white/5 backdrop-blur">
            <template #header>
              <h3 class="text-base font-semibold text-white">
                Trips for {{ selectedDate || "all dates" }}
              </h3>
            </template>

            <UTable :data="tableRows" />
          </UCard>
        </div>
      </section>

      <UCard
        v-else
        class="border border-dashed border-white/10 bg-white/5 backdrop-blur text-center text-blue-100"
      >
        <div class="space-y-4 py-12">
          <UIcon
            name="i-lucide-route"
            class="mx-auto h-10 w-10 text-primary-400"
          />
          <h3 class="text-xl font-semibold text-white">
            Upload your first trip log
          </h3>
          <p class="text-sm text-blue-200">
            Bring your MyPeugeot XLSX export to unlock a personalized map and
            insights dashboard.
          </p>
          <div class="flex justify-center gap-3">
            <UButton
              color="primary"
              icon="i-lucide-upload"
              @click="triggerFileInput"
            >
              Upload log
            </UButton>
            <UButton variant="ghost" color="primary" @click="triggerFileInput">
              Learn more
            </UButton>
          </div>
        </div>
      </UCard>
    </UContainer>
  </div>
</template>
