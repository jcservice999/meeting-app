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

  let originalText = "";
  try {
    const body = await req.json().catch(() => ({}));
    originalText = body.text || "";
    
    if (!originalText || !originalText.trim()) {
      return new Response(JSON.stringify({ result: originalText, success: true }), {
        headers: { ...corsHeaders, "Content-Type": "application/json" },
      });
    }

    const GEMINI_API_KEY = Deno.env.get("GEMINI_API_KEY");
    if (!GEMINI_API_KEY) {
      console.error("缺少 GEMINI_API_KEY");
      return new Response(JSON.stringify({ result: originalText, success: false, error: "Missing API Key" }), {
        headers: { ...corsHeaders, "Content-Type": "application/json" },
      });
    }

    console.log("收到文字，長度:", originalText.length);

    // 使用 gemini-1.5-flash 以獲得更穩定的回應
    const response = await fetch(
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${GEMINI_API_KEY}`,
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

原文：${originalText}`
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
      // 就算 API 報錯也回傳原文，不讓前端掛掉
      return new Response(JSON.stringify({ result: originalText, success: false, error: data.error?.message }), {
        headers: { ...corsHeaders, "Content-Type": "application/json" },
      });
    }

    const result = data.candidates?.[0]?.content?.parts?.[0]?.text?.trim() || originalText;
    console.log("處理成功，結果長度:", result.length);

    return new Response(JSON.stringify({ result, success: true }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });

  } catch (e) {
    console.error("Edge Function 重大錯誤:", e.message);
    return new Response(JSON.stringify({ result: originalText, success: false, error: e.message }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });
  }
});
```

## 部署步驟

1. Supabase Dashboard → Edge Functions
2. 點擊 **Create New Function**
3. 名稱：`gemini-punctuation`
4. 貼上上方程式碼
5. 點擊 **Save** 並確保已部署

## 限制

| 項目 | 免費額度 |
|------|----------|
| 每分鐘 (RPM) | 15 |
| 每日 (RPD) | 1,500 |
| 每月 | 免費 |
