<script lang="ts">
  import { onMount } from "svelte";

  import { createClient } from "@supabase/supabase-js";

  const public_api_key =
    "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImpreXB1d2JuZ2Vvc2FkdW1scHJxIiwicm9sZSI6ImFub24iLCJpYXQiOjE2OTM2ODgxNjAsImV4cCI6MjAwOTI2NDE2MH0.maQaKFE4FMG5driCzvRjCUWVPo4jrPpeNMTmV-UbPu4";
  const project_url = "https://jkypuwbngeosadumlprq.supabase.co";
  const supabase = createClient(project_url, public_api_key);

  let last_refreshed: Date | null = null;

  async function refresh(station: string) {
    let request = {
      headers: {
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Headers": "*",
      },
      body: station,
    };
    const { data, error } = await supabase.functions.invoke(
      "refresh-index",
      request
    );
  }

  interface UpcomingTrain {
    station: string;
    route_display_name: string;
    colors: string[];

    direction: string;
    projected_arrival: Date;
    status: string;
  }

  let upcoming_trains: UpcomingTrain[] = [];

  async function fetch(name: string) {
    let begin = supabase
      .from("realtime")
      .select(
        "station, route_display_name, colors, direction, projected_arrival, status"
      );

    let begin2 =
      name.toLowerCase() === "all" ? begin : begin.eq("station", name);

    const { data, error } = await begin2;

    if (data === null) {
      throw new Error("fetched data was null...");
    }

    upcoming_trains = data;
  }

  let station = "all";
  async function poll() {
    await refresh(station);
    await fetch(station);
    last_refreshed = new Date();
  }

  poll();
  setInterval(async () => {
    await poll();
  }, 30000);

  setInterval(() => {
    fetch(station);
  }, 1000);

  interface Station {
    station: string;
    original_upcoming_trains: UpcomingTrain[];
  }

  let stations: Station[] = [];

  $: {
    let aggregated_stations = new Map<string, UpcomingTrain[]>();

    for (let upcoming_train of upcoming_trains) {
      let station: string = upcoming_train.station;
      let projected_arrival: Date = new Date(upcoming_train.projected_arrival);
      upcoming_train.projected_arrival = projected_arrival;
      let prev_value = aggregated_stations.get(station);
      let new_value =
        prev_value == null ? [upcoming_train] : [...prev_value, upcoming_train];
      aggregated_stations.set(station, new_value);
    }
    let out_stations = [];
    for (let upcoming_train of upcoming_trains) {
      let station: string = upcoming_train.station;
      if (aggregated_stations.has(station)) {
        let type_script_is_a_good_programming_language =
          aggregated_stations.get(station);
        if (type_script_is_a_good_programming_language != null) {
          let new_thingy: Station = {
            station,
            original_upcoming_trains:
              type_script_is_a_good_programming_language,
          };

          out_stations.push(new_thingy);
          aggregated_stations.delete(station);
        }
      }
    }

    stations = out_stations;
  }

  function niceify_title(s: string) {
    return s
      .replaceAll("_", " ")
      .toLowerCase()
      .split(" ")
      .map((s) => s.charAt(0).toUpperCase() + s.slice(1))
      .join(" ")
      .replace("Street", "St")
      .replace("Fourteenth", "14th")
      .replace("Ninth", "9th")
      .replace("Twenty Third", "23rd")
      .replace("Thirty Third", "33rd");
  }

  let time = new Date();
  onMount(() => {
    const interval = setInterval(() => {
      time = new Date();
    }, 1000);

    return () => {
      clearInterval(interval);
    };
  });

  function prettify_eta(date: Date) {
    let time_in_minutes = (date.getTime() - time.getTime()) / (1000 * 60);

    return Math.round(Math.max(time_in_minutes, 0)).toString() + "m";
  }
</script>

<main>
  <div class="time">
    {time.toLocaleTimeString("en-us")}
  </div>
  <div class="date">
    {time.toLocaleDateString("en-us", {
      weekday: "long",
      year: "numeric",
      month: "short",
      day: "numeric",
    })}
  </div>
  {#if last_refreshed !== null}
    <div class="date">
      <span>last refreshed: </span>

      {last_refreshed.toLocaleTimeString("en-us")}
    </div>
  {/if}
  <div class="stations-grid">
    {#each stations as station}
      <div class="station-card">
        <div class="station-title">
          <div>
            <strong>{niceify_title(station.station)}</strong>
          </div>
        </div>
        <div>
          {#each station.original_upcoming_trains as upcoming_train}
            <div class="upcoming-train">
              <div class="train-title">
                <div class="line-circles">
                  {#each upcoming_train.colors as color}
                    <div
                      class="line-circle"
                      style="background-color: {color}"
                    />
                  {/each}
                </div>
                {#if upcoming_train.status === "ARRIVING_NOW"}
                  <div class="arriving-now-chip">ARRIVING NOW</div>
                {:else}
                  <span>
                    <strong>
                      {prettify_eta(upcoming_train.projected_arrival)}
                    </strong>
                  </span>
                {/if}
                <div class="route-display-name">
                  <strong>
                    {upcoming_train.route_display_name}
                  </strong>
                  <span>
                    {upcoming_train.direction.replace("TO_", "to ")}
                  </span>
                </div>
              </div>
            </div>
          {/each}
        </div>
      </div>
    {/each}
  </div>
</main>

<style>
  .station-card {
    border-radius: 8px;
    border: 4px solid black;
    font-size: 1em;
    font-weight: 500;
    font-family: inherit;
    background-color: #f9f9f9;
    transition: border-color 0.25s;
    grid-template-rows: auto auto;
    padding-left: 1rem;
    padding-right: 1rem;
    padding-bottom: 0.5rem;
  }

  main {
    display: flex;
    flex-direction: column;
    place-items: center;
    height: 100vh;
  }

  .stations-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(25rem, 1fr));
    gap: 1rem;
    width: 80%;
  }

  .station-title {
    font-size: 1.5em;
    display: flex;
    justify-content: center;
    width: 100%;
  }

  .time {
    padding-top: 1rem;
    font-size: 3.2rem;
    font-weight: bold;
  }

  .date {
    padding-bottom: 1.5rem;
  }

  .train-title {
    display: flex;
    flex-direction: row;
    gap: 0.5rem;
    align-items: center;
  }

  .line-circles {
    display: flex;
    flex-direction: row;
    min-width: 2rem;
  }

  .line-circle {
    height: 1rem;
    width: 1rem;
    border-radius: 0.5rem;
  }

  .arriving-now-chip {
    border-radius: 4px;
    padding: 0.2rem;
    border: 1px solid #2d72d2;
    color: white;
    background-color: #2d72d2;
    text-justify: center;
    text-wrap: nowrap;
  }

  @media (prefers-color-scheme: dark) {
    .station-card {
      background-color: #242424;
      border-color: rgba(255, 255, 255, 0.87);
    }
  }
</style>
