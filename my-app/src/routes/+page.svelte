
<script>
  import { onMount } from 'svelte';
  import stopsData from './stops.json';

  let stops = stopsData.data;
  let searchQuery = '';
  let filteredStops = [];
  let selectedStops = [];
  let departures = {};
  let selectedDepartures = {};
  let errors = {};

  const API_KEY = '7c63389a42094132975142d2005f23d5'; // Replace with your actual API key

  $: {
    if (searchQuery) {
      filteredStops = stops.filter(stop => 
        stop.attributes.name.toLowerCase().includes(searchQuery.toLowerCase())
      );
    } else {
      filteredStops = [];
    }
  }

  async function fetchDepartures(stopId) {
    try {
      const response = await fetch(`https://api-v3.mbta.com/predictions?filter[stop]=${stopId}&include=route,trip&api_key=${API_KEY}`);
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const data = await response.json();

      if (!data.data || data.data.length === 0) {
        departures[stopId] = [];
        errors[stopId] = "No departures found for this stop.";
        return;
      }

      departures[stopId] = data.data.map(departure => ({
        id: departure.id,
        routeId: departure.relationships.route.data.id,
        tripId: departure.relationships.trip.data.id,
        arrivalTime: departure.attributes.arrival_time,
      }));

      // Fetch route and trip information
      const routeIds = [...new Set(departures[stopId].map(d => d.routeId))];
      const tripIds = [...new Set(departures[stopId].map(d => d.tripId))];

      const [routesResponse, tripsResponse] = await Promise.all([
        fetch(`https://api-v3.mbta.com/routes?filter[id]=${routeIds.join(',')}&api_key=${API_KEY}`),
        fetch(`https://api-v3.mbta.com/trips?filter[id]=${tripIds.join(',')}&api_key=${API_KEY}`)
      ]);

      if (!routesResponse.ok || !tripsResponse.ok) {
        throw new Error('Failed to fetch route or trip information');
      }

      const [routesData, tripsData] = await Promise.all([
        routesResponse.json(),
        tripsResponse.json()
      ]);

      const routes = routesData.data.reduce((acc, route) => {
        acc[route.id] = route.attributes.short_name || route.attributes.long_name;
        return acc;
      }, {});

      const trips = tripsData.data.reduce((acc, trip) => {
        acc[trip.id] = trip.attributes.headsign;
        return acc;
      }, {});

      departures[stopId] = departures[stopId].map(departure => ({
        ...departure,
        routeName: routes[departure.routeId],
        destination: trips[departure.tripId]
      }));

      errors[stopId] = null;
    } catch (error) {
      console.error('Error fetching departures:', error);
      errors[stopId] = "Failed to fetch departures. Please try again later.";
      departures[stopId] = [];
    }

    departures = departures;
    errors = errors;
  }

  function addStop(stop) {
    selectedStops = [...selectedStops, stop];
    searchQuery = '';
    filteredStops = [];
    fetchDepartures(stop.id);
  }

  function selectDeparture(stopId, departureId) {
    selectedDepartures[stopId] = departureId;
    selectedDepartures = selectedDepartures;
  }

  function addNewStop() {
    selectedStops = [...selectedStops, null];
  }

  function getTimeUntilArrival(arrivalTime) {
    const now = new Date();
    const arrival = new Date(arrivalTime);
    const diffInMinutes = Math.round((arrival - now) / 60000);
    
    if (diffInMinutes <= 0) {
      return 'Arriving now';
    } else if (diffInMinutes === 1) {
      return '1 minute';
    } else {
      return `${diffInMinutes} minutes`;
    }
  }
</script>

<div class="route-builder">
  {#each selectedStops as stop, index}
    <div class="stop-cell">
      {#if stop}
        <h3>{stop.attributes.name}</h3>
        {#if errors[stop.id]}
          <p class="error">{errors[stop.id]}</p>
        {:else if departures[stop.id]}
          {#if departures[stop.id].length === 0}
            <p>No departures available for this stop.</p>
          {:else}
            <select on:change={(e) => selectDeparture(stop.id, e.target.value)}>
              <option value="">Select a departure</option>
              {#each departures[stop.id] as departure}
                <option value={departure.id}>
                  {departure.routeName} - {departure.destination}
                </option>
              {/each}
            </select>
            {#if selectedDepartures[stop.id]}
              {#each departures[stop.id] as departure}
                {#if departure.id === selectedDepartures[stop.id]}
                  <p>Arriving in: {getTimeUntilArrival(departure.arrivalTime)}</p>
                {/if}
              {/each}
            {/if}
          {/if}
        {:else}
          <p>Loading departures...</p>
        {/if}
      {:else}
        <input 
          bind:value={searchQuery} 
          placeholder="Search for a stop" 
          aria-label="Search for a stop"
        />
        {#if filteredStops.length > 0}
          <ul>
            {#each filteredStops as result}
              <li>
                <button on:click={() => addStop(result)}>
                  {result.attributes.name}
                </button>
              </li>
            {/each}
          </ul>
        {:else if searchQuery}
          <p>No stops found</p>
        {/if}
      {/if}
    </div>
  {/each}
  
  <button on:click={addNewStop}>+ Add Stop</button>
</div>

<style>
  .route-builder {
    max-width: 600px;
    margin: 0 auto;
  }

  .stop-cell {
    border: 1px solid #ccc;
    padding: 1rem;
    margin-bottom: 1rem;
    border-radius: 4px;
  }

  input, select {
    width: 100%;
    padding: 0.5rem;
    margin-bottom: 0.5rem;
  }

  ul {
    list-style-type: none;
    padding: 0;
  }

  li {
    margin-bottom: 0.5rem;
  }

  button {
    width: 100%;
    padding: 0.5rem;
    background-color: #007bff;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }

  button:hover {
    background-color: #0056b3;
  }

  .error {
    color: red;
    font-weight: bold;
  }
</style>