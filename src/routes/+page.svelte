<script>
	import { onMount } from 'svelte';

	// --- Existing State ---
	let csvData = [];
	let headers = [];
	let visibleColumns = {};
	let sortColumn = '';
	let sortDirection = 'asc';
	let fileInput;
	let fileName = '';
	let isLoading = false;
	let selectedTeams = []; // For 2-team comparison
	let comparisonResult = null; // For 2-team comparison

	// --- New State for Match Simulation ---
	let teamList = []; // List of teams for dropdowns
	let redTeams = [null, null, null]; // Selected teams for Red Alliance
	let blueTeams = [null, null, null]; // Selected teams for Blue Alliance
	let simulationResult = null; // Stores the results of the match simulation

	// Reactive statement to update teamList when csvData changes
	$: teamList = csvData.map(row => row.Team).filter(Boolean).sort((a, b) => {
		const numA = parseInt(a);
		const numB = parseInt(b);
		if (!isNaN(numA) && !isNaN(numB)) {
			return numA - numB;
		}
		return a.localeCompare(b);
	});

	// Helper function for dropdowns
	function getAvailableTeamsForSlot(index, allianceType) {
		const currentSelectionInThisSlot = allianceType === 'red' ? redTeams[index] : blueTeams[index];
		const allSelectedTeams = [...redTeams, ...blueTeams].filter(Boolean);
		const selectedInOtherSlots = new Set(
			allSelectedTeams.filter(team => team !== currentSelectionInThisSlot)
		);
		return teamList.filter(team => !selectedInOtherSlots.has(team));
	}


	// Handle file upload
	function handleFileUpload(event) {
		const file = event.target.files[0];
		if (!file) return;
		fileName = file.name;
		isLoading = true;
		resetDataInternal();
		const reader = new FileReader();
		reader.onload = (e) => { parseCSV(e.target.result); isLoading = false; };
		reader.onerror = () => { console.error('Error reading file'); isLoading = false; };
		reader.readAsText(file);
	}

	// Parse CSV text into data array
	function parseCSV(text) {
		const rows = text.trim().split('\n');
		const rawHeaders = rows[0].split(',');
		headers = rawHeaders.map(h => h.trim());
		visibleColumns = {};
		headers.forEach(header => { visibleColumns[header] = true; });
		csvData = rows.slice(1).map(row => {
			const values = row.split(',');
			const rowData = {};
			headers.forEach((header, index) => {
				rowData[header] = values[index] ? values[index].trim() : '';
			});
			return rowData;
		}).filter(row => row.Team);
	}

	// Toggle column visibility
	function toggleColumn(column) {
		visibleColumns = {...visibleColumns, [column]: !visibleColumns[column]};
	}

	// Sort data by column
	function sortData(column) {
		if (sortColumn === column) {
			sortDirection = sortDirection === 'asc' ? 'desc' : 'asc';
		} else {
			sortColumn = column;
			sortDirection = 'asc';
		}
		csvData = [...csvData].sort((a, b) => {
			const valueA = a[column]; const valueB = b[column];
			const numA = parseFloat(valueA); const numB = parseFloat(valueB);
			if (!isNaN(numA) && !isNaN(numB)) {
				return sortDirection === 'asc' ? numA - numB : numB - numA;
			}
			const strA = String(valueA || ''); const strB = String(valueB || '');
			return sortDirection === 'asc' ? strA.localeCompare(strB) : strB.localeCompare(strA);
		});
	}

	// Toggle team selection for 2-team comparison
	function toggleTeamSelection(team) {
		const index = selectedTeams.findIndex(t => t === team);
		if (index !== -1) {
			selectedTeams = selectedTeams.filter(t => t !== team);
		} else {
			selectedTeams = selectedTeams.length >= 2 ? [...selectedTeams.slice(1), team] : [...selectedTeams, team];
		}
		comparisonResult = selectedTeams.length === 2 ? calculateComparison() : null;
	}

	// Calculate statistical comparison between two teams (Total EPA)
	function calculateComparison() {
        const team1Data = csvData.find(row => row.Team === selectedTeams[0]);
		const team2Data = csvData.find(row => row.Team === selectedTeams[1]);
		if (!team1Data || !team2Data) return { error: "Couldn't find data for selected teams" };

		const t1EPA = safeParseFloat(team1Data.total_epa_estimate);
		const t1Std = safeParseFloat(team1Data.total_epa_std);
		const t2EPA = safeParseFloat(team2Data.total_epa_estimate);
		const t2Std = safeParseFloat(team2Data.total_epa_std);

		if (isNaN(t1EPA) || isNaN(t1Std) || isNaN(t2EPA) || isNaN(t2Std)) { // Check safeParseFloat guards
             return { error: "Invalid total EPA data for comparison" };
        }

		const meanDiff = t1EPA - t2EPA;
		const combinedVar = Math.pow(t1Std, 2) + Math.pow(t2Std, 2);
		const combinedStd = Math.sqrt(combinedVar);

		let prob1Better = 0.5;
		if (combinedStd > 0) {
            const z = meanDiff / combinedStd;
            prob1Better = normalCDF(z);
        } else if (meanDiff > 0) prob1Better = 1.0;
        else if (meanDiff < 0) prob1Better = 0.0;

		return {
			team1: selectedTeams[0], team2: selectedTeams[1],
			team1TotalEPA: t1EPA.toFixed(2), team1TotalStd: t1Std.toFixed(2),
			team2TotalEPA: t2EPA.toFixed(2), team2TotalStd: t2Std.toFixed(2),
			probabilityTeam1Better: (prob1Better * 100).toFixed(1),
			probabilityTeam2Better: ((1 - prob1Better) * 100).toFixed(1)
		};
	}

	// Standard normal cumulative distribution function approximation
	function normalCDF(x) {
        const b1 = 0.31938153, b2 = -0.356563782, b3 = 1.781477937;
        const b4 = -1.821255978, b5 = 1.330274429, p = 0.2316419, c = 0.39894228;
        if (x >= 0) { const t = 1 / (1 + p * x); return 1 - c * Math.exp(-x*x/2)*t*(t*(t*(t*(t*b5+b4)+b3)+b2)+b1); }
        else { const t = 1 / (1 - p * x); return c * Math.exp(-x*x/2)*t*(t*(t*(t*(t*b5+b4)+b3)+b2)+b1); }
	}

	// Reset all data and selections
	function resetData() {
		resetDataInternal();
		if (fileInput) fileInput.value = '';
		fileName = '';
	}

	// Internal reset helper
	function resetDataInternal() {
		csvData = []; headers = []; visibleColumns = {}; selectedTeams = [];
		comparisonResult = null; teamList = []; redTeams = [null, null, null];
		blueTeams = [null, null, null]; simulationResult = null;
	}

	// Function to get team data
	function getTeamData(teamNumber) {
		if (!teamNumber) return null;
		const data = csvData.find(row => row.Team === teamNumber);
		if (!data) { console.warn(`Data not found for team ${teamNumber}`); return null; }
		return data;
	}

	// Function to safely parse float
	function safeParseFloat(value) {
		const num = parseFloat(value); return isNaN(num) ? 0 : num;
	}

	// Calculate alliance statistics - MODIFIED
	function calculateAllianceStats(allianceTeams) {
		let TotalMatchEPA = 0, varianceTotal = 0; // For overall score prediction
		let totalEndgameEPA = 0, varianceEndgame = 0; // For endgame > 14 prob
		let totalAutoCoralEPA = 0; // For coral > 20 prob
        // varianceAutoCoralApprox will use varianceTotal as a proxy
		let probAllExit = 1.0; // For exit & coral prob
		let validDataCount = 0;

		for (const teamNum of allianceTeams) {
			const teamData = getTeamData(teamNum);
			if (teamData) {
				validDataCount++;
				const totalEPA = safeParseFloat(teamData.total_epa_estimate);
				const totalStd = safeParseFloat(teamData.total_epa_std);
				const endgameEPA = safeParseFloat(teamData.endgame_epa_estimate);
				const endgameStd = safeParseFloat(teamData.endgame_epa_std);
				const exitProb = safeParseFloat(teamData.auto_exit_prob);

				TotalMatchEPA += totalEPA;
				varianceTotal += Math.pow(totalStd, 2); // Used for overall score AND coral approx

				totalEndgameEPA += endgameEPA;
				varianceEndgame += Math.pow(endgameStd, 2);

				probAllExit *= exitProb;

				totalAutoCoralEPA += safeParseFloat(teamData.auto_coral_l1_epa_estimate);
				totalAutoCoralEPA += safeParseFloat(teamData.auto_coral_l2_epa_estimate);
				totalAutoCoralEPA += safeParseFloat(teamData.auto_coral_l3_epa_estimate);
				totalAutoCoralEPA += safeParseFloat(teamData.auto_coral_l4_epa_estimate);
			}
		}

        if (validDataCount === 0) return null; // Cannot calculate if no team data found

		const stdDevTotal = Math.sqrt(varianceTotal);
		const stdDevEndgame = Math.sqrt(varianceEndgame);
        const stdDevAutoCoralApprox = stdDevTotal; // Using total std dev as proxy

        // Prob Endgame > 14
        let probEndgameGt14 = 0;
        if (stdDevEndgame > 0) probEndgameGt14 = 1 - normalCDF((14 - totalEndgameEPA) / stdDevEndgame);
        else if (totalEndgameEPA > 14) probEndgameGt14 = 1;

        // Prob Auto Coral > 20 (using approximation)
        let probCoralGt20 = 0;
        if (stdDevAutoCoralApprox > 0) {
            // Calculate P(Score > 20) = 1 - P(Score <= 20)
             const zCoral = (20 - totalAutoCoralEPA) / stdDevAutoCoralApprox;
             probCoralGt20 = 1 - normalCDF(zCoral);
        } else if (totalAutoCoralEPA > 20) {
            probCoralGt20 = 1; // If mean > 20 and std dev is 0, prob is 1
        } // If mean <= 20 and std dev is 0, prob remains 0


        // Prob At Least One Coral (Approximation)
        const probAtLeastOneCoral = Math.min(1, totalAutoCoralEPA / 2.0); // Simple heuristic

        // Prob Exit AND Coral
        const probExitAndCoral = probAllExit * probAtLeastOneCoral;


		return {
			meanScore: TotalMatchEPA,
			stdDevScore: stdDevTotal,
            meanEndgame: totalEndgameEPA,
            stdDevEndgame: stdDevEndgame,
            probEndgameGt14: probEndgameGt14,
            probCoralGt20: probCoralGt20, // <-- ADDED
            probAllExit: probAllExit,
            probAtLeastOneCoral: probAtLeastOneCoral,
            probExitAndCoral: probExitAndCoral,
            totalAutoCoralEPA: totalAutoCoralEPA,
		};
	}

	// Simulate the match
	function simulateMatch() {
		if (!redTeams.every(Boolean) || !blueTeams.every(Boolean)) {
			simulationResult = { error: "Please select 3 teams for each alliance." }; return;
		}
        const allSelected = [...redTeams, ...blueTeams];
        if (new Set(allSelected).size !== 6) {
             simulationResult = { error: "Please ensure all 6 selected teams are unique." }; return;
        }

		const redStats = calculateAllianceStats(redTeams);
		const blueStats = calculateAllianceStats(blueTeams);

        if (!redStats || !blueStats) {
             simulationResult = { error: "Could not calculate stats. Check if team data exists for all selected teams." }; return;
        }

		// Win Confidence (Red vs Blue based on Total EPA)
		const meanDiff = redStats.meanScore - blueStats.meanScore;
		const combinedVar = Math.pow(redStats.stdDevScore, 2) + Math.pow(blueStats.stdDevScore, 2);
		const combinedStd = Math.sqrt(combinedVar);

		let probRedWin = 0.5;
        if (combinedStd > 0) probRedWin = normalCDF(meanDiff / combinedStd);
        else if (meanDiff > 0) probRedWin = 1.0;
        else if (meanDiff < 0) probRedWin = 0.0;

		simulationResult = {
			red: {
				teams: [...redTeams],
				predictedScore: redStats.meanScore,
				scoreStdDev: redStats.stdDevScore,
                probEndgameGt14: redStats.probEndgameGt14,
                probCoralGt20: redStats.probCoralGt20, // <-- Pass through result
                probExitAndCoral: redStats.probExitAndCoral,
			},
			blue: {
				teams: [...blueTeams],
				predictedScore: blueStats.meanScore,
				scoreStdDev: blueStats.stdDevScore,
                probEndgameGt14: blueStats.probEndgameGt14,
                probCoralGt20: blueStats.probCoralGt20, // <-- Pass through result
                probExitAndCoral: blueStats.probExitAndCoral,
			},
			probRedWin: probRedWin,
			probBlueWin: 1 - probRedWin,
            error: null
		};
	}

    // Clear simulation result if selections change
    $: {
        redTeams; blueTeams;
        if (simulationResult && !simulationResult.error) {
             const currentSelections = [...redTeams, ...blueTeams].sort().join(',');
             const resultSelections = simulationResult.red && simulationResult.blue ? [...simulationResult.red.teams, ...simulationResult.blue.teams].sort().join(',') : '';
             if (currentSelections !== resultSelections) {
                 simulationResult = null;
             }
        } else if (simulationResult?.error) {
             simulationResult = null;
        }
    }

</script>

<main>
	<h1>CSV Data Viewer & Analyzer</h1>

	<div class="upload-container">
		<div class="file-input-wrapper">
			<input type="file" accept=".csv" on:change={handleFileUpload} bind:this={fileInput} id="csv-upload" />
			<label for="csv-upload" class="upload-button">{fileName ? 'Change File' : 'Upload CSV File'}</label>
			{#if fileName}
				<span class="file-name">{fileName}</span>
				<button class="reset-button" on:click={resetData}>Reset All</button>
			{/if}
		</div>
	</div>

	{#if isLoading}
		<p>Loading CSV data...</p>
	{:else if headers.length > 0}
        {#if comparisonResult}
			<div class="comparison-container">
				<h3>Team Comparison - Total EPA</h3>
				{#if comparisonResult.error} <p class="error">{comparisonResult.error}</p>
				{:else}
					<div class="comparison-cards">
						<div class="team-card {comparisonResult.probabilityTeam1Better > 50 ? 'winning' : ''}">
							<h4>{comparisonResult.team1}</h4>
							<p>Total EPA: {comparisonResult.team1TotalEPA} ± {comparisonResult.team1TotalStd}</p>
							<div class="probability-bar" style="width: {comparisonResult.probabilityTeam1Better}%">{comparisonResult.probabilityTeam1Better}%</div>
						</div>
						<div class="versus">VS</div>
						<div class="team-card {comparisonResult.probabilityTeam2Better > 50 ? 'winning' : ''}">
							<h4>{comparisonResult.team2}</h4>
							<p>Total EPA: {comparisonResult.team2TotalEPA} ± {comparisonResult.team2TotalStd}</p>
							<div class="probability-bar" style="width: {comparisonResult.probabilityTeam2Better}%">{comparisonResult.probabilityTeam2Better}%</div>
						</div>
					</div>
					<p class="comparison-summary">There is a {comparisonResult.probabilityTeam1Better}% chance that {comparisonResult.team1} performs better than {comparisonResult.team2} based on Total EPA.</p>
				{/if}
			</div>
		{:else if selectedTeams.length === 1}
			<div class="selection-prompt"><p>Select one more team from the table below to compare with {selectedTeams[0]}</p></div>
		{/if}

        <div class="column-toggles">
			<h3>Toggle Table Columns:</h3>
			<div class="checkbox-container">
				{#each headers as header}
					<label><input type="checkbox" checked={visibleColumns[header]} on:change={() => toggleColumn(header)} /> {header}</label>
				{/each}
			</div>
		</div>

        <div class="table-container">
			<table>
				<thead>
					<tr>
						<th class="team-select-header">Compare (Max 2)</th>
						{#each headers as header} {#if visibleColumns[header]}
								<th on:click={() => sortData(header)}> {header} {#if sortColumn === header}<span class="sort-indicator">{sortDirection === 'asc' ? '↑' : '↓'}</span>{/if}</th>
						{/if} {/each}
					</tr>
				</thead>
				<tbody>
					{#each csvData as row (row.Team)}
						<tr class={selectedTeams.includes(row.Team) ? 'selected-row' : ''}>
							<td class="team-select-cell">
								{#if row.Team && row.Team !== 'N/A - No data available'}
									<input type="checkbox" checked={selectedTeams.includes(row.Team)} on:change={() => toggleTeamSelection(row.Team)} title="Select up to 2 teams" />
								{/if}
							</td>
							{#each headers as header} {#if visibleColumns[header]}<td>{row[header]}</td>{/if} {/each}
						</tr>
					{/each}
				</tbody>
			</table>
		</div>

        <div class="simulation-section">
            <h2>Match Simulator</h2>
            <p>Select 3 unique teams per alliance.</p>
            <div class="alliance-selection-container">
                <div class="alliance-column">
                    <h3 class="red-alliance-header">Red Alliance</h3>
                    {#each [0, 1, 2] as i}
                        <div class="team-selector">
                            <label for={`red-team-${i}`}>Robot {i + 1}:</label>
                            <select id={`red-team-${i}`} bind:value={redTeams[i]}>
                                <option value={null}>Select Team</option>
                                {#each getAvailableTeamsForSlot(i, 'red') as team (team)} <option value={team}>{team}</option> {/each}
                            </select>
                        </div>
                    {/each}
                </div>
                <div class="alliance-column">
                    <h3 class="blue-alliance-header">Blue Alliance</h3>
                     {#each [0, 1, 2] as i}
                        <div class="team-selector">
                            <label for={`blue-team-${i}`}>Robot {i + 1}:</label>
                            <select id={`blue-team-${i}`} bind:value={blueTeams[i]}>
                                <option value={null}>Select Team</option>
                                {#each getAvailableTeamsForSlot(i, 'blue') as team (team)} <option value={team}>{team}</option> {/each}
                            </select>
                        </div>
                    {/each}
                </div>
            </div>

            <button class="simulate-button" on:click={simulateMatch} disabled={!redTeams.every(Boolean) || !blueTeams.every(Boolean) || new Set([...redTeams, ...blueTeams]).size !== 6} > Simulate Match </button>

            {#if simulationResult}
                <div class="simulation-results">
                    {#if simulationResult.error}
                        <p class="error">{simulationResult.error}</p>
                    {:else}
                        <h3>Simulation Results</h3>
                        <div class="results-comparison">
                            <div class="result-card red-result">
                                <h4>Red Alliance ({simulationResult.red.teams.join(', ')})</h4>
                                <p>Predicted Score (Total EPA): {simulationResult.red.predictedScore.toFixed(2)} ± {simulationResult.red.scoreStdDev.toFixed(2)}</p>
                                <p>Barge RP Prob. {(simulationResult.red.probEndgameGt14 * 100).toFixed(1)}%</p>
                                <p>Coral RP Prob. {(simulationResult.red.probCoralGt20 * 100).toFixed(1)}%</p>  <p>Auto RP Prob {(simulationResult.red.probExitAndCoral * 100).toFixed(1)}%</p>
                                <div class="win-probability red-prob">Win Prob. {(simulationResult.probRedWin * 100).toFixed(1)}%</div>
                            </div>
                            <div class="versus-results">VS</div>
                            <div class="result-card blue-result">
                                <h4>Blue Alliance ({simulationResult.blue.teams.join(', ')})</h4>
                                <p>Predicted Score (Total EPA): {simulationResult.blue.predictedScore.toFixed(2)} ± {simulationResult.blue.scoreStdDev.toFixed(2)}</p>
                                <p>Barge RP Prob. {(simulationResult.blue.probEndgameGt14 * 100).toFixed(1)}%</p>
                                <p>Coral RP Prob. {(simulationResult.blue.probCoralGt20 * 100).toFixed(1)}%</p> <p>Auto RP Prob. {(simulationResult.blue.probExitAndCoral * 100).toFixed(1)}%</p>
                                 <div class="win-probability blue-prob">Win Probability: {(simulationResult.probBlueWin * 100).toFixed(1)}%</div>
                            </div>
                        </div>
                        <p class="result-summary">Prediction based on Total EPA suggests a <strong>{(simulationResult.probRedWin * 100).toFixed(1)}%</strong> chance for the Red Alliance to win.</p>
                    {/if}
                </div>
            {/if}
        </div> {:else if !isLoading}
		<div class="empty-state"><p>Upload a CSV file to view and analyze its contents</p></div>
	{/if}
</main>

<style>
	/* --- All existing styles remain unchanged --- */
	main { max-width: 1200px; margin: 0 auto; padding: 1rem; font-family: sans-serif; }
	.upload-container { margin-bottom: 2rem; padding: 1.5rem; border: 2px dashed #ccc; border-radius: 8px; background-color: #f9f9f9; }
	.file-input-wrapper { display: flex; align-items: center; flex-wrap: wrap; gap: 0.75rem; }
	input[type="file"] { position: absolute; width: 0; height: 0; opacity: 0; }
	.upload-button { display: inline-block; padding: 0.75rem 1.5rem; background-color: #4a86e8; color: white; border-radius: 4px; cursor: pointer; font-weight: bold; transition: background-color 0.2s; }
	.upload-button:hover { background-color: #3a76d8; }
	.file-name { font-weight: 500; }
	.reset-button { padding: 0.5rem 1rem; background-color: #f0f0f0; border: 1px solid #ccc; color: #333; border-radius: 4px; cursor: pointer; transition: background-color 0.2s; }
	.reset-button:hover { background-color: #e0e0e0; }
	.empty-state { padding: 3rem; text-align: center; background-color: #f5f5f5; border-radius: 8px; color: #666; }
	.column-toggles { margin-bottom: 1rem; padding: 1rem; background-color: #eee; border-radius: 4px; }
    .column-toggles h3 { margin-top: 0; margin-bottom: 0.5rem; }
	.checkbox-container { display: flex; flex-wrap: wrap; gap: 0.5rem 1rem; }
	.checkbox-container label { display: inline-flex; align-items: center; gap: 0.25rem; padding: 0.25rem 0.5rem; background-color: #f8f8f8; border: 1px solid #ddd; border-radius: 4px; font-size: 0.85rem; cursor: pointer; }
    .checkbox-container input { margin-right: 5px; }
	.table-container { overflow-x: auto; margin-bottom: 2rem; }
	table { width: 100%; border-collapse: collapse; font-size: 0.9rem; }
	th, td { padding: 0.6rem 0.5rem; border: 1px solid #ddd; text-align: left; white-space: nowrap; }
	th { background-color: #f2f2f2; cursor: pointer; user-select: none; position: sticky; top: 0; z-index: 1;}
	th:hover { background-color: #e0e0e0; }
	.team-select-header { width: 100px; text-align: center; font-size: 0.8rem;}
	.team-select-cell { text-align: center; }
    .team-select-cell input { cursor: pointer; }
	tr:nth-child(even) { background-color: #f9f9f9; }
	.selected-row { background-color: #e3f2fd !important; font-weight: bold; }
	.sort-indicator { margin-left: 0.25rem; font-size: 0.7rem; }
	.comparison-container { margin-bottom: 2rem; padding: 1.5rem; border-radius: 8px; background-color: #f5f5f5; box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1); }
    .comparison-container h3 { margin-top: 0; text-align: center;}
	.comparison-cards { display: flex; align-items: stretch; justify-content: space-between; gap: 1rem; margin: 1rem 0; }
	.team-card { flex: 1; padding: 1rem; border-radius: 8px; background-color: white; box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1); transition: all 0.2s; display: flex; flex-direction: column;}
	.team-card.winning { background-color: #e8f5e9; border: 2px solid #4caf50; }
    .team-card h4 { margin-top: 0; margin-bottom: 0.5rem; text-align: center; }
    .team-card p { font-size: 0.9rem; margin-bottom: auto; /* Push bar to bottom */}
	.versus { font-weight: bold; font-size: 1.5rem; color: #555; align-self: center;}
	.probability-bar { height: 24px; background-color: #4a86e8; color: white; display: flex; align-items: center; justify-content: center; border-radius: 4px; margin-top: 1rem; width: 0; min-width: 40px; font-size: 0.9rem; transition: width 0.5s ease-out; }
    .team-card.winning .probability-bar { background-color: #4caf50; }
	.comparison-summary { font-weight: 500; text-align: center; margin-top: 1rem; }
	.selection-prompt { margin: 1rem 0 2rem 0; padding: 0.75rem 1rem; background-color: #fff3e0; border-left: 4px solid #ff9800; border-radius: 4px; text-align: center; }
	.error { color: #d32f2f; font-weight: bold; text-align: center; background-color: #ffebee; padding: 0.5rem; border-radius: 4px; border: 1px solid #d32f2f;}
	.simulation-section { margin-top: 2.5rem; padding: 1.5rem; border: 1px solid #ccc; border-radius: 8px; background-color: #fafafa; }
    .simulation-section h2 { margin-top: 0; text-align: center; color: #333; margin-bottom: 0.5rem; }
    .simulation-section > p { text-align: center; color: #555; margin-bottom: 1.5rem; }
	.alliance-selection-container { display: flex; flex-wrap: wrap; justify-content: space-around; gap: 2rem; margin-bottom: 1.5rem; }
	.alliance-column { flex: 1; min-width: 250px; padding: 1rem; border-radius: 6px; background-color: #fff; box-shadow: 0 1px 3px rgba(0,0,0,0.05); }
    .alliance-column h3 { margin-top: 0; margin-bottom: 1rem; text-align: center; padding-bottom: 0.5rem; border-bottom: 2px solid; }
    .red-alliance-header { color: #e53935; border-color: #e53935; }
    .blue-alliance-header { color: #1e88e5; border-color: #1e88e5; }
	.team-selector { display: flex; align-items: center; margin-bottom: 0.75rem; gap: 0.5rem; }
    .team-selector label { width: 60px; font-size: 0.9rem; font-weight: 500; }
	.team-selector select { flex-grow: 1; padding: 0.5rem; border: 1px solid #ccc; border-radius: 4px; font-size: 0.95rem; }
    .simulate-button { display: block; width: 200px; margin: 1.5rem auto 1.5rem auto; padding: 0.8rem 1.5rem; background-color: #43a047; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 1rem; font-weight: bold; transition: background-color 0.2s, opacity 0.2s; }
    .simulate-button:hover { background-color: #388e3c; }
    .simulate-button:disabled { background-color: #bdbdbd; cursor: not-allowed; opacity: 0.7; }
    .simulation-results { margin-top: 1.5rem; padding: 1.5rem; background-color: #fff; border: 1px solid #ddd; border-radius: 6px; }
    .simulation-results h3 { margin-top: 0; text-align: center; margin-bottom: 1.5rem; color: #444; }
    .results-comparison { display: flex; align-items: stretch; justify-content: space-between; gap: 1rem; margin-bottom: 1.5rem; flex-wrap: wrap; }
    .result-card { flex: 1; min-width: 280px; padding: 1rem 1.2rem; border-radius: 6px; border: 1px solid #ccc; background-color: #f9f9f9; }
     .result-card h4 { margin-top: 0; margin-bottom: 1rem; padding-bottom: 0.5rem; border-bottom: 1px solid #eee; text-align: center; font-size: 1.1rem; }
     .result-card p { font-size: 0.9rem; margin-bottom: 0.6rem; line-height: 1.4; color: #333; }
     .red-result { border-left: 4px solid #e53935; }
     .blue-result { border-left: 4px solid #1e88e5; }
     .red-result h4 { color: #c62828; }
     .blue-result h4 { color: #1565c0; }
    .versus-results { font-weight: bold; font-size: 1.5rem; color: #555; align-self: center; padding: 0 1rem; }
    .win-probability { margin-top: 1rem; padding: 0.6rem; border-radius: 4px; text-align: center; font-weight: bold; color: white; }
    .red-prob { background-color: rgba(229, 57, 53, 0.8); }
    .blue-prob { background-color: rgba(30, 136, 229, 0.8); }
    .result-summary { text-align: center; margin-top: 1rem; font-weight: 500; font-size: 1.05rem; }
</style>