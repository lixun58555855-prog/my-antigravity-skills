---
name: xiyou-ranking-pro
description: >-
  Use this skill whenever the user provides an ASIN and a keyword to check their rankings, or explicitly asks to call Xiyou MCP (西柚mcp) to check 24-hour SP and Organic ranks. It fetches the data and renders a highly precise HTML chart.
---

# Xiyou Ranking Pro Workflow (西柚排名高精度查询)

When the user asks to check the ranking for an ASIN and a keyword, follow these exact steps to fetch the data and render the precise chart:

## 1. Prepare the Payload and Fetch Data
Call the Xiyou MCP tool to fetch hourly ranking data.
Create a JSON file (e.g., `req_body.json`) with:
```json
{
    "jsonrpc": "2.0", "id": 1234, "method": "tools/call",
    "params": {
        "name": "get_asin_keyword_rank_hourly",
        "arguments": {
            "country": "<Target Country, default UK unless specified>",
            "keyword": "<Insert Keyword>", "asin": "<Insert ASIN>", "date": "<YYYY-MM-DD (Use yesterday or today)>",
            "user_task": "调用西柚mcp查询排名", "intent_summary": "查询指定词和ASIN的24小时排名"
        }
    }
}
```
Run this curl command to fetch the data:
```powershell
curl.exe -s -X POST -H "Content-Type: application/json" --data-binary "@req_body.json" "https://mcp.xydc.com/mcp?mcp_token=mcp_ce0f6b5ecac07ee4601be3e4aa1e0658" -o rank_result.json
```

## 2. Parse the Data
Read `rank_result.json`. If the array `displayPositions` is empty for all 24 hours, inform the user that there is no data/the item is unranked.
Otherwise, extract the `totalRank` for `displayPosition == "or"` (Organic) and `sp` (Sponsored) for all 24 hours (00:00 to 23:00). If an hour is missing a rank type, use `null`.

## 3. Generate the High-Precision HTML Chart
Read the template from `./templates/chart.html` (relative to this SKILL.md file). 
Inject the following parameters into the template to generate the final HTML artifact (save it as `chart_widget.html` in the conversation's artifact directory):
1. **`[INJECT_ASIN]`**: The ASIN.
2. **`[INJECT_KEYWORD]`**: The Keyword.
3. **`[INJECT_COUNTRY]`**: The Country (e.g., UK, US).
4. **`[INJECT_ORGANIC_DATA]`**: The 24-element JSON array of organic ranks (e.g., `[3, 3, null, ...]`).
5. **`[INJECT_SP_DATA]`**: The 24-element JSON array of SP ranks.
6. **`[INJECT_MAX_RANK]`**: Calculate the maximum rank found in both arrays and add 5 to it (e.g., if max rank is 30, use 35). Set `minRank` dynamically if needed, usually `1`.

Output an `<agent-embed src="file:///<path-to-artifact>"></agent-embed>` tag in your response.
