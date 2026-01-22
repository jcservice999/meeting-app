# gemini-punctuation Edge Function

使用 Google Gemini API 為字幕添加標點符號和修正錯別字。

## Supabase Secrets 需求

確保已設定：
- `GEMINI_API_KEY` - Google AI Studio 的 API Key

## 程式碼

在 Supabase Dashboard → Edge Functions → **Create New Function** → 名稱：`gemini-punctuation`

```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
};

serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  try {
    const { text } = await req.json();
    
    if (!text || !text.trim()) {
      return new Response(JSON.stringify({ result: text || "" }), {
        headers: { ...corsHeaders, "Content-Type": "application/json" },
      });
    }

    const GEMINI_API_KEY = Deno.env.get("GEMINI_API_KEY");
    if (!GEMINI_API_KEY) {
      throw new Error("缺少 GEMINI_API_KEY");
    }

    console.log("收到文字，長度:", text.length);

    const response = await fetch(
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash-latest:generateContent?key=${GEMINI_API_KEY}`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          contents: [{
            parts: [{
              text: `你是專業的中文語音轉錄校正助手。

任務：
1. 為文字添加適當的繁體中文標點符號（句號。、逗號，、問號？、驚嘆號！等）
2. 修正明顯的同音字錯誤（如「在/再」、「他/她/它」、「的/地/得」）

規則：
- 不可刪除或增加任何詞語
- 不可改變語句順序
- 只輸出處理後的文字，不要有任何說明或解釋

原文：${text}`
            }]
          }],
          generationConfig: {
            temperature: 0.1,
            maxOutputTokens: 1000
          }
        })
      }
    );

    const data = await response.json();
    
    if (!response.ok) {
      console.error("Gemini API 錯誤:", JSON.stringify(data));
      throw new Error(data.error?.message || "Gemini API 錯誤");
    }

    const result = data.candidates?.[0]?.content?.parts?.[0]?.text?.trim() || text;
    console.log("處理成功，結果長度:", result.length);

    return new Response(JSON.stringify({ result, success: true }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });

  } catch (e) {
    console.error("Edge Function 錯誤:", e.message);
    return new Response(JSON.stringify({ error: e.message, result: "" }), {
      status: 500,
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });
  }
});
```

## 部署步驟

1. Supabase Dashboard → Edge Functions
2. 點擊 **New Function**
3. 名稱：`gemini-punctuation`
4. 貼上上方程式碼
5. 點擊 **Deploy**

## 測試

```bash
curl -X POST https://YOUR_PROJECT.supabase.co/functions/v1/gemini-punctuation \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ANON_KEY" \
  -d '{"text": "各位同學大家好今天我們來討論一下"}'
```

## 限制

| 項目 | 免費額度 |
|------|----------|
| 每分鐘 (RPM) | 15 |
| 每日 (RPD) | 1,500 |
| 每月 | 免費（無限） |
