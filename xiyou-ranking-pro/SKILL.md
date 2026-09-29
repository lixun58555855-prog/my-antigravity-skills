---
name: xiyou-ranking-pro
description: >-
  STRICT WORKFLOW: Use this skill whenever the user asks to check the Xiyou ranking (西柚排名), or provides an ASIN and a keyword. It MUST fetch a rolling 24-hour data (yesterday + today) and render a precise HTML widget. NEVER generate a table or markdown chart.
---

# Xiyou Ranking Pro Workflow (Rolling 24H)

**CRITICAL INSTRUCTION**: You are strictly forbidden from generating Markdown Tables or Mermaid charts. You MUST use the exact HTML template below. 

## Step 1: Fetch Data via MCP (Yesterday and Today)
Execute the following `curl` script exactly using the `run_command` tool. Replace `<Country>`, `<Keyword>`, `<ASIN>` with the user's request. It automatically fetches both yesterday and today to build a rolling 24-hour window.

```powershell
$country = "<Country>"
$keyword = "<Keyword>"
$asin = "<ASIN>"
$today = (Get-Date).ToString("yyyy-MM-dd")
$yest = (Get-Date).AddDays(-1).ToString("yyyy-MM-dd")

function Fetch-Rank ($date, $outfile) {
    $body = @{ jsonrpc="2.0"; id=1; method="tools/call"; params=@{ name="get_asin_keyword_rank_hourly"; arguments=@{ country=$country; keyword=$keyword; asin=$asin; date=$date } } } | ConvertTo-Json -Depth 10
    Invoke-RestMethod -Uri "https://mcp.xydc.com/mcp?mcp_token=mcp_ce0f6b5ecac07ee4601be3e4aa1e0658" -Method Post -ContentType "application/json" -Body $body -OutFile $outfile
}

Fetch-Rank $yest "rank_yest.json"
Fetch-Rank $today "rank_today.json"
```

## Step 2: Parse and Combine the JSON Data
1. Read both `rank_yest.json` and `rank_today.json`.
2. Extract the `trends` array from yesterday and the `trends` array from today.
3. Concatenate them: `combined_trends = yesterday_trends + today_trends`.
4. Take the **last 24 elements** of `combined_trends` (this is the rolling 24-hour window ending at the latest available hour).
5. Create three JavaScript arrays:
   - `organic`: The `totalRank` where `displayPosition == "or"` for these 24 items. (Use `null` if missing).
   - `sp`: The `totalRank` where `displayPosition == "sp"` for these 24 items. (Use `null` if missing).
   - `labels`: A string array for the X-axis labels. For each of the 24 items, parse the time from the `date` string (e.g. `2026-09-29T08:00:00.000+01:00`). The local hour is `08:00`. Calculate Beijing time by adding 7 hours (since +01:00 is 7 hours behind +08:00). So `08:00` becomes `15:00`. Format the label strictly as `"08:00\n15:00"` (Local time on top, Beijing time on bottom, removing any text like 'BJ' to keep it clean and prevent overlapping).

## Step 3: Generate the HTML Artifact
Create `chart_widget.html` in your artifact directory. Copy the EXACT HTML code below. 
Replace placeholders:
- `[INJECT_ASIN]`, `[INJECT_KEYWORD]`, `[INJECT_COUNTRY]`
- `[INJECT_ORGANIC_DATA]` -> The 24-element JSON array of organic ranks
- `[INJECT_SP_DATA]` -> The 24-element JSON array of SP ranks
- `[INJECT_LABELS]` -> The 24-element JSON array of string labels
- `[INJECT_MAX_RANK]` -> Max rank + 2.

```html
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <script src="https://www.gstatic.com/antigravity/web/dev/tailwindcss.min.js"></script>
</head>
<body class="bg-transparent text-[var(--foreground)] antialiased p-2">
    <div class="bg-[var(--card)] text-[var(--foreground)] border border-[var(--border)] rounded-xl p-5 shadow-sm">
        <div class="mb-4">
            <h2 class="text-[var(--foreground)] font-semibold text-lg">ASIN [INJECT_ASIN] ('[INJECT_KEYWORD]' [INJECT_COUNTRY]站) 滚动24小时排名趋势</h2>
            <p class="text-[var(--muted-foreground)] text-sm mt-1">
                以最新数据点为横轴末端，横跨完整 24 小时。横坐标上方为当地时间，下方为北京时间。无数字的点表示未查到排名。
            </p>
        </div>
        <div id="chart-container" class="relative w-full h-[360px]"></div>
        <div class="flex items-center justify-center space-x-6 mt-10 text-sm font-medium">
            <div class="flex items-center"><span class="w-3 h-3 rounded-full bg-blue-500 mr-2 shadow-sm"></span>自然排名 (Organic)</div>
            <div class="flex items-center"><span class="w-3 h-3 rounded-full bg-orange-500 mr-2 shadow-sm"></span>SP排名 (Sponsored)</div>
        </div>
    </div>
    <script>
        const organic = [INJECT_ORGANIC_DATA];
        const sp = [INJECT_SP_DATA];
        const labels = [INJECT_LABELS];
        const container = document.getElementById('chart-container');
        function draw() {
            const w = container.clientWidth || 600; const h = container.clientHeight || 360;
            const padX = 40; const padY = 20; const bottomPad = 60; 
            let allValidRanks = [...organic, ...sp].filter(x => x != null);
            if (allValidRanks.length === 0) allValidRanks = [1];
            const dataMin = Math.min(...allValidRanks);
            const maxRank = [INJECT_MAX_RANK]; 
            const minRank = Math.max(1, dataMin > 5 ? dataMin - 2 : 1);  
            const mapY = (val) => padY + ((val - minRank) / (maxRank - minRank)) * (h - padY - bottomPad);
            const mapX = (idx) => padX + (idx / 23) * (w - padX * 1.5);
            let svg = `<svg viewBox="0 0 ${w} ${h}" class="w-full h-full overflow-visible">`;
            let yStep = (maxRank - minRank) > 30 ? 5 : ((maxRank - minRank) > 15 ? 2 : 1);
            for(let r=minRank; r<=maxRank; r+=yStep) {
                let y = mapY(r);
                svg += `<line x1="${padX}" y1="${y}" x2="${w - padX/2}" y2="${y}" stroke="var(--border)" stroke-dasharray="2" opacity="0.6"/>`;
                svg += `<text x="${padX - 10}" y="${y + 4}" font-size="12" font-weight="500" fill="var(--muted-foreground)" text-anchor="end">${r}</text>`;
            }
            for(let i=0; i<24; i++) {
                let x = mapX(i); 
                let lines = (labels[i] || "").split("\\n");
                if(lines.length === 2) {
                    svg += `<text x="${x}" y="${h - 25}" font-size="10" fill="var(--muted-foreground)" text-anchor="end" transform="rotate(-45 ${x} ${h - 25})"><tspan x="${x}" dy="0">${lines[0]}</tspan><tspan x="${x}" dy="12">${lines[1]}</tspan></text>`;
                }
            }
            function drawLine(data, color, textOffsetY) {
                let path = ""; let isFirst = true;
                for(let i=0; i<24; i++) {
                    if (data[i] != null) {
                        if (isFirst) { path += `M ${mapX(i)} ${mapY(data[i])} `; isFirst = false; }
                        else { path += `L ${mapX(i)} ${mapY(data[i])} `; }
                    } else { isFirst = true; }
                }
                svg += `<path d="${path}" fill="none" stroke="${color}" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>`;
                for(let i=0; i<24; i++) {
                    if (data[i] != null) {
                        let cx = mapX(i); let cy = mapY(data[i]);
                        svg += `<circle cx="${cx}" cy="${cy}" r="4" fill="${color}" stroke="var(--card)" stroke-width="1.5" />`;
                        svg += `<text x="${cx}" y="${cy + textOffsetY}" font-size="11" font-weight="bold" fill="${color}" text-anchor="middle">${data[i]}</text>`;
                    }
                }
            }
            drawLine(organic, "#3b82f6", 14); drawLine(sp, "#f97316", -8);
            svg += `</svg>`; container.innerHTML = svg;
        }
        draw(); window.addEventListener('resize', draw);
    </script>
</body>
</html>
```

## Step 4: Display the Artifact
End your response by rendering the HTML widget inline using:
`<agent-embed src="file:///<absolute-path-to-your-chart_widget.html>"></agent-embed>`
