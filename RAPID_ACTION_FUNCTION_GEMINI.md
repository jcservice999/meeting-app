# 🚀 Supabase Edge Function：Gemini 標點與摘要 (rapid-action)

請使用以下程式碼更新您的 `rapid-action` 函數。此版本支援 **Gemini 2.0 Flash** 且針對「標點處理」模式進行了優化，能確保**不修改原文語意**。

## 完整程式碼

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
    const { conversationText, mode, model: requestedModel } = await req.json();
    
    // 從環境變數獲取 OpenRouter API Key
    const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY");
    if (!OPENROUTER_API_KEY) throw new Error("缺少 OPENROUTER_API_KEY");

    // 根據模式決定 Prompt
    let systemPrompt = "";
    let targetModel = requestedModel || "google/gemini-flash-1.5-exp";

    if (mode === "punctuation") {
      // 標點處理模式：極其嚴格，禁止改字
      systemPrompt = "你是一個專業的中文標點符號處理助手。你的任務是接收用戶傳送的原始轉錄文字，並為其補上正確的繁體中文標點符號。規則：1. 禁止修改、刪除、增加任何原本的文字內容。2. 禁止修正語法。3. 禁止進行摘要。4. 必須 100% 保留所有原始字詞。";
    } else {
      // 預設模式：會議摘要
      systemPrompt = "請根據以下會議對話內容，整理成繁體中文重點摘要，包含主要討論事項與結論。";
    }

    const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: targetModel,
        messages: [
          { role: "system", content: systemPrompt },
          { role: "user", content: conversationText }
        ],
        temperature: 0.1, // 降低隨機性，確保標點穩定
      }),
    });

    const data = await response.json();
    const resultText = data.choices[0].message.content.trim();

    return new Response(JSON.stringify({ 
      summary: resultText, // 為了相容前端，結果放在 summary
      text: resultText,
      success: true 
    }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });

  } catch (e) {
    return new Response(JSON.stringify({ error: e.message }), {
      status: 500,
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });
  }
});
```

---

## 🛠️ 更新步驟

1. 登入 [Supabase Dashboard](https://supabase.com/)
2. 前往 **Edge Functions** -> 點擊進入 `rapid-action`
3. 點擊 **Files** 尋找 `index.ts`
4. 用上方的程式碼**覆蓋**原本的所有內容件。
5. **重要**：如果您還沒有在 **Settings -> Edge Functions -> Secrets** 裡面設定 `OPENROUTER_API_KEY`，請務必現在設定。
6. 完成後點擊 **Save**。

---

## 💡 為什麼要這樣做？
- **Gemini 2.0 Flash** 是目前最快、標點最準確的免費模型。
- 這個版本加入了 `mode` 偵測，當 Whisper 伺服器傳送「標點處理」請求時，AI 會切換到**嚴禁改字**的模式，解決語意被修改的問題。
- 加入了 `temperature: 0.1` 參數，讓標點產出更穩定。
