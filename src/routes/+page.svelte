<script lang="ts">
  import { create, all, type Complex } from "mathjs";
  import { onMount } from "svelte";

  const math = create(all);

  // Custom function for parallel impedance ||
  math.import({
    parallel: function (z1: Complex | number, z2: Complex | number) {
      const num = math.multiply(z1, z2);
      const den = math.add(z1, z2);
      return math.divide(num, den);
    },
  });

  interface HistoryItem {
    id: number;
    exp: string;
    res: string;
  }

  // --- STATE ---
  let expression = $state("100 + 50j");
  let history = $state<HistoryItem[]>([]);
  let copyStatus = $state("");

  // --- SMART INPUT LOGIC ---
  function handleKeydown(e: KeyboardEvent) {
    const input = e.target as HTMLInputElement;
    const start = input.selectionStart || 0;
    const end = input.selectionEnd || 0;
    const pairs: Record<string, string> = { "(": ")", "[": "]", "{": "}" };

    if (pairs[e.key]) {
      e.preventDefault();
      expression =
        expression.slice(0, start) +
        e.key +
        pairs[e.key] +
        expression.slice(end);
      setTimeout(() => input.setSelectionRange(start + 1, start + 1), 0);
    }

    if (e.key === "Backspace" && start === end && start > 0) {
      const charBefore = expression[start - 1];
      const charAfter = expression[start];
      if (pairs[charBefore] === charAfter) {
        e.preventDefault();
        expression =
          expression.slice(0, start - 1) + expression.slice(start + 1);
        setTimeout(() => input.setSelectionRange(start - 1, start - 1), 0);
      }
    }
  }

  // --- LOGIC: EVALUATION (RE-FIXED) ---
  let evaluation = $derived.by(() => {
    try {
      if (!expression.trim()) return { res: null, err: "" };

      // 1. Handle j5, 5j, dan j (Gunakan regex yang lebih aman untuk desimal)
      let cleaned = expression
        .replace(/j(\d*\.?\d+)/g, "$1i") // j5 atau j.5 -> 5i atau .5i
        .replace(/(\d*\.?\d+)j/g, "$1i") // 5j atau .5j -> 5i atau .5i
        .replace(/j/g, "i"); // j murni -> i

      // 2. Handle operator paralel ||
      // Kita pecah manual stringnya berdasarkan || untuk menghindari Regex yang terlalu kompleks/looping
      if (cleaned.includes("||")) {
        const parts = cleaned.split("||");
        if (parts.length === 2 && parts[1].trim() !== "") {
          cleaned = `parallel(${parts[0]}, ${parts[1]})`;
        } else if (parts.length > 2) {
          // Jika ada lebih dari satu || (misal: A || B || C)
          cleaned = parts.reduce((acc, part) => `parallel(${acc}, ${part})`);
        }
      }

      // 3. Auto-fix kurung gantung
      const openBrackets = (cleaned.match(/\(/g) || []).length;
      const closeBrackets = (cleaned.match(/\)/g) || []).length;
      if (openBrackets > closeBrackets)
        cleaned += ")".repeat(openBrackets - closeBrackets);

      const evalRes = math.evaluate(cleaned);
      return { res: evalRes, err: "" };
    } catch (e: any) {
      return { res: null, err: "Sintaks Belum Lengkap" };
    }
  });

  let result = $derived(evaluation.res);
  let errorMessage = $derived(evaluation.err);

  // --- DERIVED CALCULATIONS ---
  let polar = $derived.by(() => {
    if (result && typeof result === "object" && "isComplex" in result) {
      return { r: result.abs(), theta: (result.arg() * 180) / Math.PI };
    }
    return { r: typeof result === "number" ? Math.abs(result) : 0, theta: 0 };
  });

  let displayedRect = $derived.by(() => {
    if (result && typeof result === "object" && "isComplex" in result) {
      return `${result.re.toFixed(2)} ${result.im >= 0 ? "+" : "-"} j${Math.abs(result.im).toFixed(2)}`;
    }
    return result !== null ? result.toString() : "---";
  });

  let displayedPolar = $derived(
    polar.r ? `${polar.r.toFixed(2)} ∠ ${polar.theta.toFixed(2)}°` : "---",
  );

  let vectorPos = $derived.by(() => {
    const scale = 80 / (polar.r || 1);
    const rad = (polar.theta * Math.PI) / 180;
    return {
      x: 100 + Math.cos(rad) * Math.min(polar.r * scale, 85),
      y: 100 - Math.sin(rad) * Math.min(polar.r * scale, 85),
    };
  });

  // --- ACTIONS ---
  async function copyToClipboard(text: string) {
    if (text === "---") return;
    await navigator.clipboard.writeText(text);
    copyStatus = "Copied!";
    setTimeout(() => (copyStatus = ""), 2000);
  }

  function saveToHistory() {
    if (result && !errorMessage) {
      history = [
        { id: Date.now(), exp: expression, res: displayedPolar },
        ...history,
      ].slice(0, 10);
      localStorage.setItem("phasor_history", JSON.stringify(history));
    }
  }

  onMount(() => {
    const saved = localStorage.getItem("phasor_history");
    if (saved) history = JSON.parse(saved);
  });
</script>

<div
  class="min-h-screen bg-[#F8FAFC] text-[#1E293B] p-4 md:p-8 font-sans transition-all"
>
  <div class="max-w-4xl mx-auto">
    <header
      class="flex items-center justify-between mb-8 border-b border-slate-200 pb-6"
    >
      <h1
        class="text-3xl font-black tracking-tighter text-blue-600 italic uppercase"
      >
        PHASOR<span class="text-slate-400 text-2xl tracking-normal italic"
          >FLOW</span
        >
      </h1>
      {#if copyStatus}
        <div
          class="bg-blue-600 text-white text-[10px] px-3 py-1 rounded-md font-bold uppercase"
        >
          {copyStatus}
        </div>
      {/if}
    </header>

    <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
      <div class="lg:col-span-2 space-y-6">
        <div class="bg-white p-6 rounded-3xl shadow-sm border border-slate-100">
          <label
            for="expression-input"
            class="text-[10px] uppercase font-black text-slate-400 mb-3 block tracking-widest"
            >Expression Input</label
          >
          <input
            id="expression-input"
            bind:value={expression}
            onkeydown={handleKeydown}
            spellcheck="false"
            autocomplete="off"
            class="w-full bg-slate-50 p-5 rounded-2xl border-2 {errorMessage
              ? 'border-red-50 text-red-400'
              : 'border-transparent focus:border-blue-600'} outline-none text-2xl font-mono shadow-inner transition-all"
          />
          <div class="mt-4 flex gap-2">
            <button
              onclick={saveToHistory}
              class="bg-blue-600 text-white text-[10px] font-black py-3 px-6 rounded-xl active:scale-95 transition-all shadow-lg shadow-blue-100 uppercase tracking-widest"
              >Save Result</button
            >
            <button
              onclick={() => (expression = "")}
              class="bg-slate-100 text-slate-500 text-[10px] font-black py-3 px-6 rounded-xl hover:bg-slate-200 transition-colors uppercase tracking-widest"
              >Reset</button
            >
          </div>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
          <button
            onclick={() => copyToClipboard(displayedRect)}
            class="bg-white p-6 rounded-3xl border border-slate-200 text-left hover:border-blue-300 transition-all active:scale-95 group shadow-sm"
          >
            <span
              class="text-[10px] uppercase font-black text-slate-400 flex justify-between tracking-widest"
              >Rectangular <span>(Copy)</span></span
            >
            <div
              class="mt-2 text-2xl font-mono font-bold text-slate-800 tracking-tight group-hover:text-blue-600 leading-tight"
            >
              {displayedRect}
            </div>
          </button>
          <button
            onclick={() => copyToClipboard(displayedPolar)}
            class="bg-white p-6 rounded-3xl border border-slate-200 text-left hover:border-blue-300 transition-all active:scale-95 group shadow-sm"
          >
            <span
              class="text-[10px] uppercase font-black text-blue-500 flex justify-between tracking-widest"
              >Polar <span>(Copy)</span></span
            >
            <div
              class="mt-2 text-2xl font-mono font-bold text-blue-600 tracking-tight leading-tight"
            >
              {displayedPolar}
            </div>
          </button>
        </div>

        {#if result && typeof result === "object" && "isComplex" in result}
          <div
            class="bg-white p-8 rounded-3xl shadow-xl border-l-4 border-blue-600 space-y-6"
          >
            <h3
              class="text-xs font-black uppercase tracking-widest text-blue-600 italic"
            >
              Penyelesaian Langkah demi Langkah:
            </h3>

            <div
              class="space-y-4 font-mono text-sm leading-relaxed text-slate-600"
            >
              <div class="flex gap-4">
                <span
                  class="flex-none w-6 h-6 bg-blue-100 text-blue-600 rounded-full flex items-center justify-center font-bold text-xs italic"
                  >1</span
                >
                <div>
                  <p class="font-bold text-slate-800 mb-1 tracking-tight">
                    Cari Magnitude (r)
                  </p>
                  <div
                    class="bg-slate-50 p-3 rounded-xl border border-slate-100 text-blue-800 font-bold overflow-x-auto shadow-inner"
                  >
                    r = √({result.re.toFixed(2)}² + {result.im.toFixed(2)}²)
                    <br />
                    r =
                    <span
                      class="text-blue-600 underline font-black tracking-widest"
                      >{polar.r.toFixed(4)}</span
                    >
                  </div>
                </div>
              </div>

              <div class="flex gap-4 border-t border-slate-50 pt-4">
                <span
                  class="flex-none w-6 h-6 bg-blue-100 text-blue-600 rounded-full flex items-center justify-center font-bold text-xs italic"
                  >2</span
                >
                <div>
                  <p class="font-bold text-slate-800 mb-1 tracking-tight">
                    Tentukan Sudut Fase (θ)
                  </p>
                  <div
                    class="bg-slate-50 p-3 rounded-xl border border-slate-100 text-blue-800 font-bold overflow-x-auto shadow-inner text-sm"
                  >
                    θ = atan2({result.im.toFixed(2)}, {result.re.toFixed(2)})
                    <br />
                    θ =
                    <span class="text-blue-600 underline font-black"
                      >{polar.theta.toFixed(2)}°</span
                    >
                  </div>
                  <div
                    class="mt-3 flex gap-2 flex-wrap text-[10px] font-bold uppercase"
                  >
                    <span class="text-slate-500 bg-slate-100 px-2 py-1 rounded"
                      >Posisi: {result.re >= 0 ? "Kanan" : "Kiri"} - {result.im >=
                      0
                        ? "Atas"
                        : "Bawah"}</span
                    >
                    <span
                      class="{result.im > 0
                        ? 'bg-orange-500 text-white'
                        : 'bg-blue-500 text-white'} px-2 py-1 rounded shadow-sm"
                    >
                      {result.im > 0
                        ? "Induktif"
                        : result.im < 0
                          ? "Kapasitif"
                          : "Resistif"}
                    </span>
                  </div>
                </div>
              </div>

              <div class="flex gap-4 border-t border-slate-50 pt-4">
                <span
                  class="flex-none w-6 h-6 bg-blue-600 text-white rounded-full flex items-center justify-center font-bold text-xs italic shadow-md"
                  >3</span
                >
                <div class="w-full">
                  <p
                    class="font-bold text-slate-800 mb-2 tracking-tight italic"
                  >
                    Hasil Akhir (Phasor)
                  </p>
                  <div
                    class="text-xl bg-blue-600 text-white p-5 rounded-2xl font-black text-center shadow-lg shadow-blue-100 tracking-widest border-b-4 border-blue-800"
                  >
                    {displayedPolar}
                  </div>
                </div>
              </div>
            </div>
          </div>
        {/if}

        <div class="bg-white p-6 rounded-3xl border border-slate-100 shadow-sm">
          <span
            class="text-[10px] uppercase font-black text-slate-400 mb-4 block tracking-widest underline decoration-slate-200 underline-offset-4"
            >Recent Memory</span
          >
          <div class="space-y-2">
            {#each history as item (item.id)}
              <button
                onclick={() => (expression = item.exp)}
                class="w-full flex justify-between items-center p-4 rounded-2xl bg-slate-50 hover:bg-blue-50 transition-all border border-transparent hover:border-blue-100 group"
              >
                <span
                  class="text-sm font-mono font-bold text-slate-600 group-hover:text-blue-600"
                  >{item.exp}</span
                >
                <span class="text-xs font-mono font-black text-slate-400"
                  >{item.res}</span
                >
              </button>
            {:else}
              <p
                class="text-center py-6 text-[10px] text-slate-300 font-black italic uppercase tracking-widest"
              >
                No history recorded
              </p>
            {/each}
          </div>
        </div>
      </div>

      <div
        class="bg-white p-8 rounded-3xl shadow-sm border border-slate-200 flex flex-col items-center sticky top-8 h-fit"
      >
        <span
          class="text-[10px] uppercase font-black text-slate-400 mb-8 tracking-[0.2em] italic"
          >Phasor Map</span
        >
        <div class="relative w-full aspect-square max-w-60">
          <svg viewBox="0 0 200 200" class="w-full h-full drop-shadow-2xl">
            <circle
              cx="100"
              cy="100"
              r="80"
              fill="none"
              stroke="#F1F5F9"
              stroke-width="2"
            />
            <circle
              cx="100"
              cy="100"
              r="40"
              fill="none"
              stroke="#F1F5F9"
              stroke-width="1"
            />
            <line
              x1="10"
              y1="100"
              x2="190"
              y2="100"
              stroke="#CBD5E1"
              stroke-width="2"
            />
            <line
              x1="100"
              y1="10"
              x2="100"
              y2="190"
              stroke="#CBD5E1"
              stroke-width="2"
            />
            <text
              x="182"
              y="112"
              class="text-[8px] fill-slate-400 font-black tracking-tighter italic"
              >RE</text
            >
            <text
              x="105"
              y="15"
              class="text-[8px] fill-slate-400 font-black tracking-tighter italic"
              >IM</text
            >
            {#if result}
              <defs>
                <marker
                  id="arrow"
                  viewBox="0 0 10 10"
                  refX="5"
                  refY="5"
                  markerWidth="4"
                  markerHeight="4"
                  orient="auto-start-reverse"
                >
                  <path d="M 0 0 L 10 5 L 0 10 z" fill="#2563eb" />
                </marker>
              </defs>
              <line
                x1="100"
                y1="100"
                x2={vectorPos.x}
                y2={vectorPos.y}
                stroke="#2563eb"
                stroke-width="3"
                stroke-linecap="round"
                marker-end="url(#arrow)"
                class="transition-all duration-300 ease-out"
              />
            {/if}
          </svg>
        </div>
        <div class="mt-8 grid grid-cols-2 gap-2 w-full text-center font-mono">
          <div class="bg-blue-50 p-4 rounded-2xl border border-blue-100">
            <div class="text-[8px] text-blue-400 font-black uppercase">
              Angle
            </div>
            <div
              class="text-xl font-black text-blue-600 tracking-tighter leading-none"
            >
              {polar.theta.toFixed(1)}°
            </div>
          </div>
          <div class="bg-slate-50 p-4 rounded-2xl border border-slate-100">
            <div class="text-[8px] text-slate-400 font-black uppercase">
              Mag
            </div>
            <div
              class="text-xl font-black text-slate-600 tracking-tighter leading-none"
            >
              {polar.r.toFixed(1)}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
