<script lang="ts">
  import Chart from 'chart.js/auto';
  import { onMount } from 'svelte';
  
  // Estado para almacenar los datos de Pokémon
  let pokemonData: { id: number; name: string; url: string }[] = [];
  let selectedPokemon = [];
  let chartInstance = null;
  let searchTerm = '';
  let loading = false;
  
  // Colores para la gráfica
  const chartColors = [
    'rgba(255, 99, 132, 0.7)',
    'rgba(54, 162, 235, 0.7)',
    'rgba(255, 206, 86, 0.7)',
    'rgba(75, 192, 192, 0.7)',
    'rgba(153, 102, 255, 0.7)',
    'rgba(255, 159, 64, 0.7)'
  ];
  
  // Define la interfaz para los datos de Pokémon
  interface PokemonAPIResponse {
    name: string;
    url: string;
  }

  // Obtener la lista de Pokémon
  async function fetchPokemonList() {
    loading = true;
    try {
      const response = await fetch('https://pokeapi.co/api/v2/pokemon?limit=151');
      const data = await response.json();
      pokemonData = data.results.map((pokemon: PokemonAPIResponse, index: number) => ({
        id: index + 1,
        name: pokemon.name,
        url: pokemon.url
      }));
    } catch (error) {
      console.error('Error al obtener la lista de Pokémon:', error);
    } finally {
      loading = false;
    }
  }
  
  // Obtener detalles de un Pokémon específico
  async function fetchPokemonDetails(pokemon) {
    try {
      const response = await fetch(pokemon.url);
      const data = await response.json();
      return {
        name: pokemon.name,
        stats: data.stats.map(stat => ({
          name: stat.stat.name,
          value: stat.base_stat
        }))
      };
    } catch (error) {
      console.error(`Error al obtener detalles de ${pokemon.name}:`, error);
      return null;
    }
  }
  
  // Actualizar la gráfica con los Pokémon seleccionados
  async function updateChart() {
    if (selectedPokemon.length === 0) return;
    
    loading = true;
    
    // Obtener detalles de cada Pokémon seleccionado
    const pokemonDetailsPromises = selectedPokemon.map(pokemon => fetchPokemonDetails(pokemon));
    const pokemonDetails = await Promise.all(pokemonDetailsPromises);
    
    // Preparar datos para la gráfica
    const chartData = {
      labels: ['HP', 'Ataque', 'Defensa', 'Ataque Esp.', 'Defensa Esp.', 'Velocidad'],
      datasets: pokemonDetails.map((pokemon, index) => ({
        label: pokemon.name.charAt(0).toUpperCase() + pokemon.name.slice(1),
        data: pokemon.stats.map(stat => stat.value),
        backgroundColor: chartColors[index % chartColors.length],
        borderColor: chartColors[index % chartColors.length].replace('0.7', '1'),
        borderWidth: 1
      }))
    };
    
    // Crear o actualizar la gráfica
    const ctx = document.getElementById('pokemonChart');
    
    if (chartInstance) {
      chartInstance.destroy();
    }
    
    chartInstance = new Chart(ctx, {
      type: 'radar',
      data: chartData,
      options: {
        scales: {
          r: {
            beginAtZero: true,
            max: 150
          }
        }
      }
    });
    
    loading = false;
  }
  
  // Filtrar Pokémon por término de búsqueda
  $: filteredPokemon = pokemonData.filter(pokemon => 
    pokemon.name.toLowerCase().includes(searchTerm.toLowerCase())
  );
  
  // Seleccionar un Pokémon
  function togglePokemon(pokemon) {
    const index = selectedPokemon.findIndex(p => p.id === pokemon.id);
    if (index >= 0) {
      selectedPokemon = [...selectedPokemon.slice(0, index), ...selectedPokemon.slice(index + 1)];
    } else {
      if (selectedPokemon.length < 6) {
        selectedPokemon = [...selectedPokemon, pokemon];
      } else {
        alert('Puedes seleccionar un máximo de 6 Pokémon para comparar');
      }
    }
    updateChart();
  }
  
  // Comprobar si un Pokémon está seleccionado
  function isPokemonSelected(pokemon) {
    return selectedPokemon.some(p => p.id === pokemon.id);
  }
  
  // Inicializar la aplicación al montar
  onMount(() => {
    fetchPokemonList();
  });
</script>

<main>
  <h1>Comparador de Estadísticas de Pokémon</h1>
  
  <div class="chart-container">
    {#if loading}
      <p class="loading">Cargando datos...</p>
    {:else if selectedPokemon.length === 0}
      <p class="no-data">Selecciona al menos un Pokémon para ver sus estadísticas</p>
    {:else}
      <canvas id="pokemonChart"></canvas>
    {/if}
  </div>
  
  <div class="controls">
    <div class="search-box">
      <input 
        type="text" 
        bind:value={searchTerm} 
        placeholder="Buscar Pokémon..."
      />
    </div>
    
    
    <div class="selected-pokemon">
      <h3>Pokémon seleccionados ({selectedPokemon.length}/6):</h3>
      <div class="selected-list">
        {#each selectedPokemon as pokemon}
          <div class="pokemon-tag" style="background-color: {chartColors[selectedPokemon.indexOf(pokemon) % chartColors.length]}">
            {pokemon.name}
            <button class="remove-btn" on:click={() => togglePokemon(pokemon)}>×</button>
          </div>
        {/each}
      </div>
    </div>
    
    <div class="pokemon-list">
      <h3>Lista de Pokémon:</h3>
      <div class="list-container">
        {#each filteredPokemon as pokemon}
          <div 
            class="pokemon-item" 
            class:selected={isPokemonSelected(pokemon)}
            on:click={() => togglePokemon(pokemon)}
          >
            <img 
              src={`https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/${pokemon.id}.png`} 
              alt={pokemon.name} 
            />
            <span>{pokemon.name}</span>
          </div>
        {/each}
      </div>
    </div>
  </div>
</main>

<style>
  main {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
    font-family: Arial, sans-serif;
  }
  
  h1 {
    text-align: center;
    color: #e53935;
    margin-bottom: 30px;
  }
  
  .chart-container {
    background-color: white;
    border-radius: 8px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    padding: 20px;
    margin-bottom: 30px;
    min-height: 400px;
    display: flex;
    justify-content: center;
    align-items: center;
  }
  
  .loading, .no-data {
    text-align: center;
    color: #757575;
    font-style: italic;
  }
  
  .controls {
    display: grid;
    grid-template-columns: 1fr;
    gap: 20px;
  }
  
  .search-box input {
    width: 100%;
    padding: 10px;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 16px;
  }
  
  .selected-pokemon {
    background-color: #f5f5f5;
    border-radius: 8px;
    padding: 15px;
  }
  
  .selected-list {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    margin-top: 10px;
  }
  
  .pokemon-tag {
    padding: 5px 10px;
    border-radius: 20px;
    color: white;
    display: flex;
    align-items: center;
    gap: 5px;
  }
  
  .remove-btn {
    background: none;
    border: none;
    color: white;
    cursor: pointer;
    font-size: 18px;
    line-height: 1;
    padding: 0;
    margin-left: 5px;
  }
  
  .pokemon-list {
    background-color: #f5f5f5;
    border-radius: 8px;
    padding: 15px;
  }
  
  .list-container {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
    gap: 10px;
    max-height: 400px;
    overflow-y: auto;
    padding: 10px 0;
  }
  
  .pokemon-item {
    background-color: white;
    border-radius: 8px;
    padding: 10px;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 5px;
    cursor: pointer;
    transition: transform 0.2s, box-shadow 0.2s;
  }
  
  .pokemon-item:hover {
    transform: translateY(-3px);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  }
  
  .pokemon-item.selected {
    border: 2px solid #e53935;
    background-color: #ffebee;
  }
  
  .pokemon-item img {
    width: 70px;
    height: 70px;
  }
  
  .pokemon-item span {
    text-transform: capitalize;
    font-size: 14px;
    text-align: center;
  }
  
  @media (min-width: 768px) {
    .controls {
      grid-template-columns: 1fr 2fr;
    }
    
    .search-box {
      grid-column: 1 / -1;
    }
  }
</style>