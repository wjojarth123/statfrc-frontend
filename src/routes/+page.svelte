<script>
	import { onMount } from 'svelte';

	// State for the CSV data
	let csvData = [];
	let headers = [];
	let visibleColumns = {};
	let sortColumn = '';
	let sortDirection = 'asc';
	let fileInput;
	let fileName = '';
	let isLoading = false;
	
	// Team comparison state
	let selectedTeams = [];
	let comparisonResult = null;
	
	// Handle file upload
	function handleFileUpload(event) {
		const file = event.target.files[0];
		if (!file) return;
		
		fileName = file.name;
		isLoading = true;
		
		const reader = new FileReader();
		
		reader.onload = (e) => {
			const text = e.target.result;
			parseCSV(text);
			isLoading = false;
		};
		
		reader.onerror = () => {
			console.error('Error reading file');
			isLoading = false;
		};
		
		reader.readAsText(file);
	}

	// Parse CSV text into data array
	function parseCSV(text) {
		const rows = text.trim().split('\n');
		headers = rows[0].split(',');
		
		// Initialize all columns as visible
		headers.forEach(header => {
			visibleColumns[header] = true;
		});
		
		// Parse data rows
		csvData = rows.slice(1).map(row => {
			const values = row.split(',');
			const rowData = {};
			
			headers.forEach((header, index) => {
				rowData[header] = values[index];
			});
			
			return rowData;
		});
		
		// Reset team selection when loading new data
		selectedTeams = [];
		comparisonResult = null;
	}

	// Toggle column visibility
	function toggleColumn(column) {
		visibleColumns[column] = !visibleColumns[column];
	}

	// Sort data by column
	function sortData(column) {
		if (sortColumn === column) {
			// Toggle sort direction if clicking the same column
			sortDirection = sortDirection === 'asc' ? 'desc' : 'asc';
		} else {
			// Set new sort column and default to ascending
			sortColumn = column;
			sortDirection = 'asc';
		}
		
		// Sort the data
		csvData = [...csvData].sort((a, b) => {
			const valueA = a[column];
			const valueB = b[column];
			
			// Handle numeric values
			const numA = parseFloat(valueA);
			const numB = parseFloat(valueB);
			
			if (!isNaN(numA) && !isNaN(numB)) {
				return sortDirection === 'asc' ? numA - numB : numB - numA;
			}
			
			// Handle string values
			if (sortDirection === 'asc') {
				return valueA.localeCompare(valueB);
			} else {
				return valueB.localeCompare(valueA);
			}
		});
	}
	
	// Toggle team selection with a limit of 2 teams
	function toggleTeamSelection(team) {
		const index = selectedTeams.findIndex(t => t === team);
		
		if (index !== -1) {
			// If team is already selected, remove it
			selectedTeams = selectedTeams.filter(t => t !== team);
			comparisonResult = null;
		} else {
			// If trying to select a third team, remove the oldest selection
			if (selectedTeams.length >= 2) {
				selectedTeams = [...selectedTeams.slice(1), team];
			} else {
				selectedTeams = [...selectedTeams, team];
			}
			
			// If we have exactly 2 teams selected, calculate comparison
			if (selectedTeams.length === 2) {
				calculateComparison();
			} else {
				comparisonResult = null;
			}
		}
	}
	
	// Calculate statistical comparison between two teams
	function calculateComparison() {
		const team1Data = csvData.find(row => row.Team === selectedTeams[0]);
		const team2Data = csvData.find(row => row.Team === selectedTeams[1]);
		
		if (!team1Data || !team2Data) {
			comparisonResult = {
				error: "Couldn't find data for selected teams"
			};
			return;
		}
		
		// Extract teleop EPA values and standard deviations
		const team1TeleopEPA = parseFloat(team1Data.teleop_epa_estimate);
		const team1TeleopStd = parseFloat(team1Data.teleop_epa_std);
		const team2TeleopEPA = parseFloat(team2Data.teleop_epa_estimate);
		const team2TeleopStd = parseFloat(team2Data.teleop_epa_std);
		
		// Check if values are valid numbers
		if (isNaN(team1TeleopEPA) || isNaN(team1TeleopStd) || isNaN(team2TeleopEPA) || isNaN(team2TeleopStd)) {
			comparisonResult = {
				error: "Invalid teleop EPA data for comparison"
			};
			return;
		}
		
		// Calculate the probability that team1 outperforms team2
		// Using the formula for the difference of two normally distributed random variables
		const meanDifference = team1TeleopEPA - team2TeleopEPA;
		const combinedVariance = Math.pow(team1TeleopStd, 2) + Math.pow(team2TeleopStd, 2);
		const combinedStdDev = Math.sqrt(combinedVariance);
		
		// Calculate z-score
		const zScore = meanDifference / combinedStdDev;
		
		// Convert z-score to probability using the standard normal CDF approximation
		const probability = normalCDF(zScore);
		
		comparisonResult = {
			team1: selectedTeams[0],
			team2: selectedTeams[1],
			team1TeleopEPA: team1TeleopEPA.toFixed(2),
			team1TeleopStd: team1TeleopStd.toFixed(2),
			team2TeleopEPA: team2TeleopEPA.toFixed(2),
			team2TeleopStd: team2TeleopStd.toFixed(2),
			probabilityTeam1Better: (probability * 100).toFixed(1),
			probabilityTeam2Better: ((1 - probability) * 100).toFixed(1)
		};
	}
	
	// Standard normal cumulative distribution function approximation
	function normalCDF(x) {
		// Constants for the approximation
		const b1 = 0.31938153;
		const b2 = -0.356563782;
		const b3 = 1.781477937;
		const b4 = -1.821255978;
		const b5 = 1.330274429;
		const p = 0.2316419;
		const c = 0.39894228;
		
		if (x >= 0) {
			const t = 1.0 / (1.0 + p * x);
			return 1.0 - c * Math.exp(-x * x / 2.0) * t * (t * (t * (t * (t * b5 + b4) + b3) + b2) + b1);
		} else {
			const t = 1.0 / (1.0 - p * x);
			return c * Math.exp(-x * x / 2.0) * t * (t * (t * (t * (t * b5 + b4) + b3) + b2) + b1);
		}
	}
	
	// Reset the data
	function resetData() {
		csvData = [];
		headers = [];
		visibleColumns = {};
		fileName = '';
		selectedTeams = [];
		comparisonResult = null;
		if (fileInput) fileInput.value = '';
	}
</script>

<main>
	<h1>CSV Data Viewer</h1>
	
	<div class="upload-container">
		<div class="file-input-wrapper">
			<input 
				type="file" 
				accept=".csv" 
				on:change={handleFileUpload}
				bind:this={fileInput}
				id="csv-upload"
			/>
			<label for="csv-upload" class="upload-button">
				{fileName ? 'Change File' : 'Upload CSV File'}
			</label>
			{#if fileName}
				<span class="file-name">{fileName}</span>
				<button class="reset-button" on:click={resetData}>Reset</button>
			{/if}
		</div>
	</div>
	
	{#if isLoading}
		<p>Loading CSV data...</p>
	{:else if headers.length > 0}
		<!-- Team comparison results -->
		{#if comparisonResult}
			<div class="comparison-container">
				<h3>Team Comparison - Teleop EPA</h3>
				{#if comparisonResult.error}
					<p class="error">{comparisonResult.error}</p>
				{:else}
					<div class="comparison-cards">
						<div class="team-card {comparisonResult.probabilityTeam1Better > 50 ? 'winning' : ''}">
							<h4>{comparisonResult.team1}</h4>
							<p>Teleop EPA: {comparisonResult.team1TeleopEPA} ± {comparisonResult.team1TeleopStd}</p>
							<div class="probability-bar" style="width: {comparisonResult.probabilityTeam1Better}%">
								{comparisonResult.probabilityTeam1Better}%
							</div>
						</div>
						<div class="versus">VS</div>
						<div class="team-card {comparisonResult.probabilityTeam2Better > 50 ? 'winning' : ''}">
							<h4>{comparisonResult.team2}</h4>
							<p>Teleop EPA: {comparisonResult.team2TeleopEPA} ± {comparisonResult.team2TeleopStd}</p>
							<div class="probability-bar" style="width: {comparisonResult.probabilityTeam2Better}%">
								{comparisonResult.probabilityTeam2Better}%
							</div>
						</div>
					</div>
					<p class="comparison-summary">
						There is a {comparisonResult.probabilityTeam1Better}% chance that {comparisonResult.team1} will perform better than {comparisonResult.team2} in teleop.
					</p>
				{/if}
			</div>
		{:else if selectedTeams.length === 1}
			<div class="selection-prompt">
				<p>Select one more team to compare with {selectedTeams[0]}</p>
			</div>
		{/if}
		
		<div class="column-toggles">
			<h3>Toggle Columns:</h3>
			<div class="checkbox-container">
				{#each headers as header}
					<label>
						<input
							type="checkbox"
							checked={visibleColumns[header]}
							on:change={() => toggleColumn(header)}
						/>
						{header}
					</label>
				{/each}
			</div>
		</div>
		
		<div class="table-container">
			<table>
				<thead>
					<tr>
						<th class="team-select-header">Compare</th>
						{#each headers as header}
							{#if visibleColumns[header]}
								<th on:click={() => sortData(header)}>
									{header}
									{#if sortColumn === header}
										<span class="sort-indicator">{sortDirection === 'asc' ? '↑' : '↓'}</span>
									{/if}
								</th>
							{/if}
						{/each}
					</tr>
				</thead>
				<tbody>
					{#each csvData as row}
						<tr class={selectedTeams.includes(row.Team) ? 'selected-row' : ''}>
							<td class="team-select-cell">
								{#if row.Team && row.Team !== 'N/A - No data available'}
									<input 
										type="checkbox" 
										checked={selectedTeams.includes(row.Team)} 
										on:change={() => toggleTeamSelection(row.Team)}
									/>
								{/if}
							</td>
							{#each headers as header}
								{#if visibleColumns[header]}
									<td>{row[header]}</td>
								{/if}
							{/each}
						</tr>
					{/each}
				</tbody>
			</table>
		</div>
	{:else}
		<div class="empty-state">
			<p>Upload a CSV file to view and analyze its contents</p>
		</div>
	{/if}
</main>

<style>
	main {
		max-width: 1200px;
		margin: 0 auto;
		padding: 1rem;
	}
	
	.upload-container {
		margin-bottom: 2rem;
		padding: 1.5rem;
		border: 2px dashed #ccc;
		border-radius: 8px;
		background-color: #f9f9f9;
	}
	
	.file-input-wrapper {
		display: flex;
		align-items: center;
		flex-wrap: wrap;
		gap: 0.75rem;
	}
	
	/* Hide the default file input */
	input[type="file"] {
		position: absolute;
		width: 0;
		height: 0;
		opacity: 0;
	}
	
	/* Custom upload button */
	.upload-button {
		display: inline-block;
		padding: 0.75rem 1.5rem;
		background-color: #4a86e8;
		color: white;
		border-radius: 4px;
		cursor: pointer;
		font-weight: bold;
		transition: background-color 0.2s;
	}
	
	.upload-button:hover {
		background-color: #3a76d8;
	}
	
	.file-name {
		font-weight: 500;
	}
	
	.reset-button {
		padding: 0.5rem 1rem;
		background-color: #f0f0f0;
		border: none;
		border-radius: 4px;
		cursor: pointer;
		transition: background-color 0.2s;
	}
	
	.reset-button:hover {
		background-color: #e0e0e0;
	}
	
	.empty-state {
		padding: 3rem;
		text-align: center;
		background-color: #f5f5f5;
		border-radius: 8px;
		color: #666;
	}
	
	.column-toggles {
		margin-bottom: 1rem;
	}
	
	.checkbox-container {
		display: flex;
		flex-wrap: wrap;
		gap: 0.5rem;
		margin-bottom: 1rem;
	}
	
	label {
		display: flex;
		align-items: center;
		gap: 0.25rem;
		padding: 0.25rem 0.5rem;
		background-color: #f0f0f0;
		border-radius: 4px;
	}
	
	.table-container {
		overflow-x: auto;
	}
	
	table {
		width: 100%;
		border-collapse: collapse;
		font-size: 0.9rem;
	}
	
	th, td {
		padding: 0.5rem;
		border: 1px solid #ddd;
		text-align: left;
	}
	
	th {
		background-color: #f2f2f2;
		cursor: pointer;
		user-select: none;
	}
	
	th:hover {
		background-color: #e0e0e0;
	}
	
	.team-select-header {
		width: 80px;
		text-align: center;
	}
	
	.team-select-cell {
		text-align: center;
	}
	
	tr:nth-child(even) {
		background-color: #f9f9f9;
	}
	
	tr:hover {
		background-color: #f0f0f0;
	}
	
	.selected-row {
		background-color: #e3f2fd !important;
	}
	
	.sort-indicator {
		margin-left: 0.25rem;
	}
	
	.comparison-container {
		margin-bottom: 2rem;
		padding: 1.5rem;
		border-radius: 8px;
		background-color: #f5f5f5;
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
	}
	
	.comparison-cards {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 1rem;
		margin: 1rem 0;
	}
	
	.team-card {
		flex: 1;
		padding: 1rem;
		border-radius: 8px;
		background-color: white;
		box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
		transition: all 0.2s;
	}
	
	.team-card.winning {
		background-color: #e8f5e9;
		border-left: 4px solid #4caf50;
	}
	
	.versus {
		font-weight: bold;
		font-size: 1.2rem;
	}
	
	.probability-bar {
		height: 24px;
		background-color: #4a86e8;
		color: white;
		display: flex;
		align-items: center;
		justify-content: center;
		border-radius: 4px;
		margin-top: 0.5rem;
		min-width: 40px;
		font-size: 0.8rem;
		transition: width 0.5s ease-out;
	}
	
	.comparison-summary {
		font-weight: 500;
		text-align: center;
		margin-top: 1rem;
	}
	
	.selection-prompt {
		margin-bottom: 1rem;
		padding: 0.75rem;
		background-color: #fff3e0;
		border-left: 4px solid #ff9800;
		border-radius: 4px;
	}
	
	.error {
		color: #d32f2f;
		font-weight: 500;
	}
</style>
