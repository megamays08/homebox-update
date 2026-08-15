<script setup lang="ts">
  import Subtitle from "~/components/global/Subtitle.vue";
  import BaseCard from "@/components/Base/Card.vue";
  import { MaintenanceFilterStatus } from "~~/lib/api/types/data-contracts";

  const api = useUserApi();

  // Look-ahead window (days). User-configurable from the card header.
  const windowDays = ref(30);
  const windowOptions = [30, 60, 90];

  type Row = {
    itemId: string;
    itemName: string;
    label: string;
    date: string; // YYYY-MM-DD
    kind: "expiration" | "maintenance";
  };

  function daysUntil(dateStr: string): number {
    const today = new Date();
    today.setHours(0, 0, 0, 0);
    const d = new Date(`${dateStr}T00:00:00`);
    return Math.round((d.getTime() - today.getTime()) / 86_400_000);
  }

  const { data: rows } = useAsyncData<Row[]>(
    "coming-due",
    async () => {
      // Date-typed custom fields (window + deadline logic applied server-side)
      // and scheduled-but-not-completed maintenance, merged into one feed.
      const [expiring, maintenance] = await Promise.all([
        api.items.fields.expiring(windowDays.value),
        api.maintenance.getAll({ status: MaintenanceFilterStatus.MaintenanceFilterStatusScheduled }),
      ]);

      const out: Row[] = [];

      for (const e of expiring.data ?? []) {
        out.push({ itemId: e.id, itemName: e.name, label: e.fieldName, date: e.date, kind: "expiration" });
      }

      for (const m of maintenance.data ?? []) {
        // scheduledDate is a date-only "YYYY-MM-DD" string over the wire.
        const sd = typeof m.scheduledDate === "string" ? m.scheduledDate : "";
        // Keep scheduled tasks that are due within the window or overdue.
        if (!sd || daysUntil(sd) > windowDays.value) {
          continue;
        }
        out.push({ itemId: m.itemID, itemName: m.itemName, label: m.name, date: sd, kind: "maintenance" });
      }

      // YYYY-MM-DD sorts correctly as a plain string (soonest / most-overdue first).
      out.sort((a, b) => a.date.localeCompare(b.date));
      return out;
    },
    { watch: [windowDays] }
  );

  function relativeLabel(n: number): string {
    if (n < 0) {
      return `${Math.abs(n)} day${Math.abs(n) === 1 ? "" : "s"} overdue`;
    }
    if (n === 0) {
      return "due today";
    }
    if (n === 1) {
      return "due tomorrow";
    }
    return `in ${n} days`;
  }

  // Urgency drives the pill colour; theme-aware for light/dark.
  function urgencyClass(n: number): string {
    if (n < 0) {
      return "bg-red-100 text-red-700 dark:bg-red-950 dark:text-red-300";
    }
    if (n <= 7) {
      return "bg-amber-100 text-amber-800 dark:bg-amber-950 dark:text-amber-300";
    }
    return "bg-muted text-muted-foreground";
  }

  function kindClass(kind: Row["kind"]): string {
    return kind === "maintenance"
      ? "bg-sky-100 text-sky-700 dark:bg-sky-950 dark:text-sky-300"
      : "bg-violet-100 text-violet-700 dark:bg-violet-950 dark:text-violet-300";
  }
</script>

<template>
  <section>
    <div class="flex items-center justify-between gap-2">
      <Subtitle>Coming Due</Subtitle>
      <div class="flex overflow-hidden rounded-md border text-xs">
        <button
          v-for="opt in windowOptions"
          :key="opt"
          class="px-2.5 py-1 transition-colors"
          :class="windowDays === opt ? 'bg-primary text-primary-foreground' : 'hover:bg-muted'"
          @click="windowDays = opt"
        >
          {{ opt }}d
        </button>
      </div>
    </div>

    <BaseCard>
      <p v-if="!rows || rows.length === 0" class="px-4 py-3 text-sm text-muted-foreground">
        Nothing due in the next {{ windowDays }} days.
      </p>
      <ul v-else class="divide-y">
        <li
          v-for="(row, i) in rows"
          :key="`${row.itemId}-${i}`"
          class="flex items-center justify-between gap-3 px-4 py-3"
        >
          <div class="min-w-0">
            <NuxtLink :to="`/item/${row.itemId}`" class="block truncate font-medium hover:underline">
              {{ row.itemName }}
            </NuxtLink>
            <span class="flex items-center gap-1.5 text-xs text-muted-foreground">
              <span
                class="rounded px-1.5 py-0.5 text-[10px] font-medium uppercase tracking-wide"
                :class="kindClass(row.kind)"
              >
                {{ row.kind === "maintenance" ? "Maint" : "Exp" }}
              </span>
              {{ row.label }} · {{ row.date }}
            </span>
          </div>
          <span
            class="shrink-0 rounded-full px-2.5 py-0.5 text-xs font-medium"
            :class="urgencyClass(daysUntil(row.date))"
          >
            {{ relativeLabel(daysUntil(row.date)) }}
          </span>
        </li>
      </ul>
    </BaseCard>
  </section>
</template>
