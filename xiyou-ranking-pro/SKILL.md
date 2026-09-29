---
name: xiyou-ranking-pro
description: >-
  STRICT WORKFLOW: Use this skill whenever the user asks to check the Xiyou ranking (西柚排名), or provides an ASIN and a keyword. It MUST fetch hourly data and render a precise HTML widget. NEVER generate a table or markdown chart.
---

# Xiyou Ranking Pro Workflow

**CRITICAL INSTRUCTION**: You are strictly forbidden from generating Markdown Tables or Mermaid charts for this task. You MUST use the exact HTML template provided below. If you deviate, the user will be extremely upset.

## Step 1: Fetch Data via MCP
Execute the following `curl` command exactly using the `run_command` tool. Replace `<Country>`, `<Keyword>`, `<ASIN>`, and `<Date>` (use today's or yesterday's date, format YYYY-MM-DD):

```powershell
$body = @{
    jsonrpc = "2.0"
    id = 1234
    method = "tools/call"
    params = @{
        name = "get_asin_keyword_rank_hourly"
        arguments = @{
            country = "<Country>"
            keyword = "<Keyword>"
            asin = "<ASIN>"
            date = "<Date>"
            user_task = "调用西柚mcp查询排名"
            intent_summary = "查询指定词和ASIN的24小时排名"
        }
    }
} | ConvertTo-Json -Depth 10

Invoke-RestMethod -Uri "https://mcp.xydc.com/mcp?mcp_token=mcp_ce0f6b5ecac07ee4601be3e4aa1e0658" -Method Post -ContentType "application/json" -Body $body -OutFile "rank_result.json"
```

## Step 2: Parse the JSON Data
Read `rank_result.json`. 
Extract the `displayPositions` for all 24 hours (from 00:00 to 23:00).
Create two JavaScript arrays:
1. `organic`: The `totalRank` where `displayPosition == "or"`. If missing, use `null`. (e.g., `[3, 3, null, 5, ...]`)
2. `sp`: The `totalRank` where `displayPosition == "sp"`. If missing, use `null`.

## Step 3: Generate the HTML Artifact
You MUST create an artifact file named `chart_widget.html` in your artifact directory.
Copy the EXACT HTML code below. Do NOT alter the CSS or SVG logic.
Replace the 6 uppercase placeholders:
- `[INJECT_ASIN]` -> The ASIN
- `[INJECT_KEYWORD]` -> The Keyword
- `[INJECT_COUNTRY]` -> The Country Code
- `[INJECT_ORGANIC_DATA]` -> The JSON array of organic ranks (e.g., `[3, 4, null...]`)
- `[INJECT_SP_DATA]` -> The JSON array of SP ranks (e.g., `[null, 2, 2...]`)
- `[INJECT_MAX_RANK]` -> Calculate the maximum rank found across both arrays, and add 2 to it. If the highest rank number is 32, replace with 34.

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
            <h2 class="text-[var(--foreground)] font-semibold text-lg">ASIN [INJECT_ASIN] ('[INJECT_KEYWORD]' [INJECT_COUNTRY]站) 24小时排名趋势</h2>
            <p class="text-[var(--muted-foreground)] text-sm mt-1">
                纵轴精确到每一个排名数字，横轴精确到每一个小时。<strong>每个数据点上的数字已直接标出</strong>以便于精准比对。无连线及无数字的点表示该时段未查到对应排名。
            </p>
        </div>
        <div id="chart-container" class="relative w-full h-[320px]"></div>
        <div class="flex items-center justify-center space-x-6 mt-8 text-sm font-medium">
            <div class="flex items-center"><span class="w-3 h-3 rounded-full bg-blue-500 mr-2 shadow-sm"></span>自然排名 (Organic)</div>
            <div class="flex items-center"><span class="w-3 h-3 rounded-full bg-orange-500 mr-2 shadow-sm"></span>SP排名 (Sponsored)</div>
        </div>
    </div>
    <script>
        const organic = [INJECT_ORGANIC_DATA];
        const sp = [INJECT_SP_DATA];
        const container = document.getElementById('chart-container');
        function draw() {
            const w = container.clientWidth || 600; const h = container.clientHeight || 320;
            const padX = 40; const padY = 20; const bottomPad = 40; 
            let allValidRanks = [...organic, ...sp].filter(x => x !== null);
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
                let x = mapX(i); let hourStr = i.toString().padStart(2, '0') + ":00";
                svg += `<text x="${x}" y="${h - 10}" font-size="10" fill="var(--muted-foreground)" text-anchor="end" transform="rotate(-45 ${x} ${h - 10})">${hourStr}</text>`;
            }
            function drawLine(data, color, textOffsetY) {
                let path = ""; let isFirst = true;
                for(let i=0; i<24; i++) {
                    if (data[i] !== null) {
                        if (isFirst) { path += `M ${mapX(i)} ${mapY(data[i])} `; isFirst = false; }
                        else { path += `L ${mapX(i)} ${mapY(data[i])} `; }
                    } else { isFirst = true; }
                }
                svg += `<path d="${path}" fill="none" stroke="${color}" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>`;
                for(let i=0; i<24; i++) {
                    if (data[i] !== null) {
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
Do NOT output a summary markdown table of the ranks.
