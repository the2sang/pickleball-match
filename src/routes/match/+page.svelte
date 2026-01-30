<script>
    import { onMount } from 'svelte';

    let names = [];
    let fixedPairs = [];
    let hours = 1;
    let pairStartRound = 1; // 🎯 파트너 고정 시작 라운드 설정
    let matches = [];
    let newName = "";
    let selectedForPair = [];
    let editingIndex = null;
    let tableRef;

    onMount(() => {
        const params = new URLSearchParams(window.location.search);
        const shareData = params.get('share');
        if (shareData) {
            try {
                const decoded = JSON.parse(atob(shareData));
                names = decoded.names || [];
                fixedPairs = decoded.fixedPairs || [];
                matches = decoded.matches || [];
                pairStartRound = decoded.pairStartRound || 1;
            } catch (e) { console.error("로드 실패", e); }
        }
    });

    const addName = () => {
        if (newName && names.length < 12) {
            names = [...names, newName.trim()];
            newName = "";
        }
    };

    const removeName = (index) => {
        const nameToRemove = names[index];
        names = names.filter((_, i) => i !== index);
        fixedPairs = fixedPairs.filter(pair => !pair.includes(nameToRemove));
    };

    const togglePairSelection = (name) => {
        if (selectedForPair.includes(name)) {
            selectedForPair = selectedForPair.filter(n => n !== name);
        } else if (selectedForPair.length < 2) {
            selectedForPair = [...selectedForPair, name];
        }
    };

    const makePair = () => {
        if (selectedForPair.length === 2) {
            fixedPairs = [...fixedPairs, [...selectedForPair]];
            selectedForPair = [];
        }
    };

    const generateSchedule = () => {
        if (names.length < 4) return alert("최소 4명의 선수가 필요합니다.");
        const matchDuration = 10;
        const totalMatchesCount = Math.floor((hours * 60) / matchDuration);
        let tempMatches = [];
        let playCount = {};
        names.forEach(n => playCount[n] = 0);

        for (let i = 1; i <= totalMatchesCount; i++) {
            let units = [];
            // 🎯 특정 라운드 이후에만 파트너십 적용
            const isPairingActive = i >= pairStartRound;
            const currentPairedNames = isPairingActive ? fixedPairs.flat() : [];

            if (isPairingActive) {
                fixedPairs.forEach(pair => {
                    units.push({ names: pair, count: Math.max(playCount[pair[0]], playCount[pair[1]]), size: 2 });
                });
            }

            names.filter(n => !currentPairedNames.includes(n)).forEach(n => {
                units.push({ names: [n], count: playCount[n], size: 1 });
            });

            units.sort((a, b) => a.count - b.count || Math.random() - 0.5);

            let selectedPlayers = [];
            let usedUnits = [];
            for (let unit of units) {
                if (selectedPlayers.length + unit.size <= 4) {
                    selectedPlayers.push(...unit.names);
                    unit.names.forEach(n => playCount[n]++);
                    usedUnits.push(unit);
                }
                if (selectedPlayers.length === 4) break;
            }

            let teamA = [], teamB = [];
            if (usedUnits[0]?.size === 2) {
                teamA = [...usedUnits[0].names];
                teamB = usedUnits.slice(1).flatMap(u => u.names);
            } else if (usedUnits.length > 1 && usedUnits[1]?.size === 2) {
                teamB = [...usedUnits[1].names];
                teamA = [usedUnits[0].names[0], ...usedUnits.slice(2).flatMap(u => u.names)];
            } else {
                teamA = [selectedPlayers[0], selectedPlayers[1]];
                teamB = [selectedPlayers[2], selectedPlayers[3]];
            }
            tempMatches.push({ round: i, teamA, teamB, isPairingActive });
        }
        matches = tempMatches;
    };

    const shareViaLink = async () => {
        const data = btoa(JSON.stringify({ names, fixedPairs, matches, pairStartRound }));
        const url = `${window.location.origin}${window.location.pathname}?share=${data}`;
        if (navigator.share) await navigator.share({ title: '피클볼 대진표', url });
        else { await navigator.clipboard.writeText(url); alert("링크 복사 완료!"); }
    };
</script>

<div class="max-w-4xl mx-auto p-4 md:p-10 bg-slate-50 min-h-screen text-slate-900 font-sans">
    <header class="bg-white p-8 rounded-3xl shadow-sm border border-slate-200 mb-8">
        <h1 class="text-2xl font-black tracking-tighter text-slate-800 mb-6">🏓 PICKLEBALL STRATEGY</h1>

        <div class="grid md:grid-cols-2 gap-8">
            <div class="space-y-4">
                <div class="flex gap-2">
                    <input bind:value={newName} placeholder="선수 이름" class="flex-1 px-4 py-2 bg-slate-100 rounded-xl outline-none" on:keydown={(e) => e.key === 'Enter' && addName()} />
                    <button on:click={addName} class="bg-blue-600 text-white px-4 py-2 rounded-xl font-bold">추가</button>
                </div>
                <div class="flex flex-wrap gap-2">
                    {#each names as name, i}
                        <button on:click={() => togglePairSelection(name)} class="px-3 py-1.5 rounded-lg text-sm font-bold border transition {selectedForPair.includes(name) ? 'bg-blue-600 text-white' : 'bg-white text-slate-600'}">{name}</button>
                    {/each}
                </div>

                {#if fixedPairs.length > 0}
                    <div class="mt-4 p-4 bg-indigo-50 rounded-2xl border border-indigo-100">
                        <label class="block text-[10px] font-black text-indigo-400 uppercase mb-3" for="start-round">파트너 고정 시작 라운드</label>
                        <div class="flex items-center gap-4">
                            <input type="range" min="1" max={Math.floor((hours * 60) / 10)} bind:value={pairStartRound} class="flex-1 accent-indigo-600" id="start-round" />
                            <span class="text-xl font-black text-indigo-600 w-8">{pairStartRound}R</span>
                        </div>
                        <p class="text-[10px] text-indigo-400 mt-1">* {pairStartRound}라운드부터 ❤️ 팀워크가 적용됩니다.</p>
                    </div>
                {/if}

                {#if selectedForPair.length === 2}
                    <button on:click={makePair} class="w-full py-2 bg-rose-500 text-white rounded-xl font-bold text-sm shadow-lg animate-pulse">🤝 파트너로 묶기</button>
                {/if}
            </div>

            <div class="flex flex-col justify-end gap-4">
                <select bind:value={hours} class="w-full p-4 bg-slate-100 rounded-2xl font-bold outline-none cursor-pointer border-none">
                    {#each [1,2,3,4,5] as h}<option value={h}>{h}시간 ({h * 6}경기)</option>{/each}
                </select>
                <button on:click={generateSchedule} class="w-full bg-slate-900 text-white font-black py-4 rounded-2xl shadow-xl hover:bg-black transition">대진표 생성하기</button>
            </div>
        </div>
    </header>

    {#if matches.length > 0}
        <div class="flex justify-end gap-2 mb-4">
            <button on:click={shareViaLink} class="px-4 py-2 bg-yellow-400 text-yellow-900 rounded-xl text-xs font-bold shadow-sm">💬 카톡 공유</button>
        </div>

        <div bind:this={tableRef} class="bg-white rounded-[2rem] shadow-2xl border border-slate-200 overflow-hidden">
            <table class="w-full border-collapse">
                <thead class="bg-slate-900 text-white text-[9px] font-black uppercase tracking-[0.2em]">
                <tr>
                    <th class="py-4 px-4 text-center w-20 border-r border-slate-800">Round</th>
                    <th class="py-4 px-4 text-center">Team A</th>
                    <th class="py-4 px-4 text-center w-10 text-slate-500 font-normal">vs</th>
                    <th class="py-4 px-4 text-center">Team B</th>
                </tr>
                </thead>
                <tbody class="divide-y divide-slate-100">
                {#each matches as match}
                    <tr class="hover:bg-slate-50/50 transition">
                        <td class="py-6 px-4 text-center border-r border-slate-50">
                            <span class="font-black text-xl {match.isPairingActive ? 'text-indigo-600' : 'text-slate-200'} italic">#{match.round}</span>
                            {#if match.isPairingActive}<span class="block text-[8px] font-bold text-indigo-400 mt-1 uppercase">Fixed</span>{/if}
                        </td>
                        <td class="py-6 px-2 text-center">
                            <div class="flex justify-center items-center gap-x-3">
                                <span class="text-lg font-extrabold text-indigo-600">{match.teamA[0]}</span>
                                {#if match.teamA[1]}<span class="text-rose-400 text-sm">{match.isPairingActive ? '❤️' : '•'}</span><span class="text-lg font-extrabold text-indigo-600">{match.teamA[1]}</span>{/if}
                            </div>
                        </td>
                        <td class="py-6 text-center text-[9px] font-black text-slate-200 tracking-tighter">VS</td>
                        <td class="py-6 px-2 text-center">
                            <div class="flex justify-center items-center gap-x-3">
                                <span class="text-lg font-extrabold text-emerald-600">{match.teamB[0]}</span>
                                {#if match.teamB[1]}<span class="text-rose-400 text-sm">{match.isPairingActive ? '❤️' : '•'}</span><span class="text-lg font-extrabold text-emerald-600">{match.teamB[1]}</span>{/if}
                            </div>
                        </td>
                    </tr>
                {/each}
                </tbody>
            </table>
        </div>
    {/if}
</div>