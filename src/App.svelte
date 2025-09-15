<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";

  import page0 from "./data/ome-zarr-page0.json";
  import page1 from "./data/ome-zarr-page1.json";
  import page2 from "./data/ome-zarr-page2.json";
  import page3 from "./data/ome-zarr-page3.json";

  const BASE = "https://forum.image.sc";
  const pages = [page0, page1, page2, page3];

  let topicsCount = 0;
  let peopleCount = 0;
  let leaderboard = [];
  let months = [];
  let longData = [];
  let iconByUser = {};

  let svgContainer;
  let width = 800, height = 450, margin = { top: 40, right: 80, bottom: 50, left: 150 };

  let playing = false;
  let timer;
   
  function avatarUrl(u) {
    const t = u?.avatar_template;
    if (!t) return null;
    let url = t.replace("{size}", "64");
    if (url.startsWith("/")) url = BASE + url;
    return url;
  }

  // === Data prep ===
  const userById = new Map();
  for (const p of pages) {
    for (const u of p.users ?? []) {
      userById.set(u.id, { username: u.username, avatar: avatarUrl(u) });
    }
  }

  const topics = pages.flatMap((p) => p.topic_list?.topics ?? []);
  const monthUserCounts = new Map();
  const userTotals = new Map();
  const seenUsers = new Set();

  for (const t of topics) {
    const iso = t.created_at || t.last_posted_at;
    if (!iso) continue;
    const month = new Date(iso).toISOString().slice(0, 7);

    if (!monthUserCounts.has(month)) monthUserCounts.set(month, new Map());
    const bucket = monthUserCounts.get(month);

    for (const p of t.posters ?? []) {
      const id = p.user_id ?? p.user?.id;
      const u = userById.get(id);
      const name = u?.username ?? (id ? `user-${id}` : "unknown");

      bucket.set(name, (bucket.get(name) || 0) + 1);
      userTotals.set(name, (userTotals.get(name) || 0) + 1);
      if (u?.username) seenUsers.add(u.username);
    }
  }

  topicsCount = topics.length;
  peopleCount = seenUsers.size;

  for (const [, u] of userById.entries()) {
    if (u.username && u.avatar) iconByUser[u.username] = u.avatar;
  }

  months = [...monthUserCounts.keys()].sort();
  let currentIndex = months.length - 1;
  const cumulative = new Map();
  longData = [];

  months.forEach((m) => {
    const bucket = monthUserCounts.get(m);
    for (const [name, value] of bucket.entries()) {
      cumulative.set(name, (cumulative.get(name) || 0) + value);
    }
    for (const [name, total] of cumulative.entries()) {
      longData.push({ date: `${m}-01`, name, value: total, icon: iconByUser[name] });
    }
  });

  leaderboard = [...userTotals.entries()].sort((a, b) => b[1] - a[1]);

  // === D3 chart ===
  const color = d3.scaleOrdinal(d3.schemeTableau10);

  onMount(() => {
    drawChart();
    updateChart(currentIndex);
  });

  function drawChart() {
    const svg = d3.select(svgContainer)
      .attr("viewBox", [0, 0, width, height]);

    svg.append("g").attr("class", "x-axis")
      .attr("transform", `translate(0,${height - margin.bottom})`);

    svg.append("g").attr("class", "y-axis")
      .attr("transform", `translate(${margin.left},0)`);

    svg.append("text")
      .attr("class", "date-label")
      .attr("x", width - margin.right)
      .attr("y", margin.top - 10)
      .attr("text-anchor", "end")
      .style("font-size", "18px")
      .style("fill", "#444");
  }

function updateChart(index) {
  const svg = d3.select(svgContainer);

  const month = months[index];
  let frame = longData.filter(d => d.date.startsWith(month));
  frame = frame.sort((a, b) => b.value - a.value).slice(0, 12);

  // Pad with placeholders if fewer than 12
  while (frame.length < 12) {
    frame.push({ name: "", value: 0, icon: "" });
  }

  const avatarSize = 24;   // fixed px size for avatars
  const labelPadding = 6;  // gap between text and avatar
  const barOffset = 80;    // horizontal offset for bars

  const x = d3.scaleLinear()
    .domain([0, d3.max(frame, d => d.value)]).nice()
    .range([margin.left + barOffset, width - margin.right]);

  const y = d3.scaleBand()
    .domain(frame.map(d => d.name))
    .range([margin.top, height - margin.bottom])
    .padding(0.2);

  const t = svg.transition().duration(300);

  // === Bars ===
  const bars = svg.selectAll(".bar").data(frame, d => d.name);

  bars.join(
    enter => enter.append("rect")
      .attr("class", "bar")
      .attr("x", margin.left + barOffset)
      .attr("y", d => y(d.name))
      .attr("height", y.bandwidth())
      .attr("width", d => x(d.value) - (margin.left + barOffset))
      .attr("fill", d => d.name ? color(d.name) : "#e5e7eb"),
    update => update.transition(t)
      .attr("y", d => y(d.name))
      .attr("height", y.bandwidth())
      .attr("width", d => x(d.value) - (margin.left + barOffset)),
    exit => exit.remove()
  );

  // === Avatars + Names ===
  const names = svg.selectAll(".name-label").data(frame, d => d.name);

  const gEnter = names.enter().append("g")
    .attr("class", "name-label")
    .attr("transform", d => `translate(${margin.left},${y(d.name)})`);

gEnter.append("a")
  .attr("class", "user-link")
  .attr("href", d => d.name ? `${BASE}/u/${d.name}` : null)
  .attr("target", "_blank")
  .append("text")
    .attr("class", "user-label")
    .attr("text-anchor", "end")
    .attr("dy", "0.35em")
    .text(d => d.name);

  gEnter.append("image")
    .attr("class", "avatar")
    .attr("href", d => d.icon || "")
    .attr("clip-path", "circle(50%)");

  // Update group position
  names.merge(gEnter).transition(t)
    .attr("transform", d => `translate(${margin.left},${y(d.name)})`);

names.merge(gEnter).select("a")
  .attr("href", d => d.name ? `${BASE}/u/${d.name}` : null);

names.merge(gEnter).select("text")
  .transition(t)
  .attr("x", 0)
  .attr("y", y.bandwidth() / 2)
  .text(d => d.name);


  // Update avatar
  names.merge(gEnter).select("image")
    .transition(t)
    .attr("x", labelPadding)
    .attr("y", y.bandwidth() / 2 - avatarSize / 2)
    .attr("width", avatarSize)
    .attr("height", avatarSize);

  names.exit().remove();

  // === Counters ===
  const counters = svg.selectAll(".counter").data(frame, d => d.name);

  counters.join(
    enter => enter.append("text")
      .attr("class", "counter")
      .attr("x", d => x(d.value) + 5)
      .attr("y", d => y(d.name) + y.bandwidth() / 2)
      .attr("dy", "0.35em")
      .text(d => d.value || ""),
    update => update.transition(t)
      .attr("x", d => x(d.value) + 5)
      .attr("y", d => y(d.name) + y.bandwidth() / 2)
      .text(d => d.value || ""),
    exit => exit.remove()
  );

  // === Axes and date label ===
  svg.select(".x-axis")
    .transition(t)
    .call(d3.axisBottom(x).ticks(width / 80));

  svg.select(".y-axis").selectAll("*").remove();

  svg.select(".date-label").text(month);
}

  // Slider controls
  function onSlider(e) {
    currentIndex = +e.target.value;
    updateChart(currentIndex);
  }

  function togglePlay() {
    playing = !playing;
    if (playing) {
      timer = setInterval(() => {
        currentIndex++;
        if (currentIndex >= months.length) currentIndex = 0;
        updateChart(currentIndex);
      }, 300);
    } else {
      clearInterval(timer);
    }
  }
</script>

<main>
  <header>
    <h1>OME-Zarr posters in the image.sc forum</h1>
    <p>Topics started up to that month in the <a href="https://forum.image.sc/" target="_blank">image.sc</a> forum with the <a href="https://forum.image.sc/tag/ome-zarr" target="_blank">ome-zarr</a> tag for which the person engaged at least once</p>
    <p class="meta">
      <span>{topicsCount} topics</span>
      <span>•</span>
      <span>{peopleCount} people</span>
    </p>
  </header>

  <section class="race-wrap">
    <svg bind:this={svgContainer}></svg>
    <div class="controls">
      <button on:click={togglePlay}>{playing ? "Pause" : "Play"}</button>
      <input type="range" min="0" max={months.length - 1} bind:value={currentIndex} on:input={onSlider} />
      <span>{months[currentIndex]}</span>
    </div>
  </section>

  <section class="board">
    <h2>Top Posters (by topic appearances)</h2>
    <ol>
      {#each leaderboard as [user, count], i}
        <li>
          <a class="user" href={`${BASE}/u/${user}`} target="_blank" rel="noopener">
            <img src={iconByUser[user] || ""} alt="" />
            <span class="rank">#{i + 1}</span>
            <span class="name">{user}</span>
          </a>
          <span class="count">{count}</span>
        </li>
      {/each}
    </ol>
  </section>
</main>

<style>
  :global(html, body) { margin: 0; background: #fafafa; }
  main { margin: 24px auto; padding: 0 18px; color:#1f2937; }
  header { text-align: center; margin-bottom: 12px; }
  h1 { margin: 0 0 6px; }
  .meta { opacity:.7; display:flex; gap:.5rem; justify-content:center; }

  .race-wrap { background:#fff; border:1px solid #e5e7eb; border-radius:16px; padding:12px;
    box-shadow:0 1px 3px rgba(0,0,0,.06); margin:18px 0 28px; }

  svg { width:100%; height:450px; }
  svg text { font-family: system-ui, sans-serif; font-size: 13px; fill: #374151; }
  .counter { font-weight: 600; fill: #111; }

  .controls { margin-top:10px; display:flex; align-items:center; gap:10px; }
  button { padding:4px 12px; border-radius:6px; border:1px solid #d1d5db; background:#f9fafb; cursor:pointer; }
  button:hover { background:#f3f4f6; }
  input[type=range] { flex:1; }

  .board { margin: 8px auto 28px; max-width: 560px; }
  .board ol { list-style:none; padding:0; }
  .board li {
    display:grid; grid-template-columns: 1fr auto; align-items:center;
    gap:12px; padding:10px 12px; background:#fff; border:1px solid #e5e7eb; border-radius:12px;
    margin:8px 0; box-shadow:0 1px 2px rgba(0,0,0,.04);
  }
  .board .user { display:flex; align-items:center; gap:10px; text-decoration:none; color:#0f62fe; }
  .board img { width:28px; height:28px; border-radius:50%; }

  .user-link {
  cursor: pointer;
  text-decoration: none;
}
.user-link:hover text {
  fill: #0f62fe; /* same blue as dashboard */
}

</style>
