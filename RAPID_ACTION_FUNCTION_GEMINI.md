# 🚀 Supabase Edge Function：Gemini 標點與摘要 (rapid-action)

請使用以下程式碼更新您的 `rapid-action` 函數。此版本加入了**完整的錯誤處理**，能幫助診斷問題。

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
    if (!OPENROUTER_API_KEY) {
      console.error("缺少 OPENROUTER_API_KEY");
      throw new Error("缺少 OPENROUTER_API_KEY，請在 Supabase Secrets 中設定");
    }

    console.log("收到請求，mode:", mode, "text長度:", conversationText?.length);

    // 根據模式決定 Prompt
    let systemPrompt = "";
    let targetModel = requestedModel || "meta-llama/llama-3.2-3b-instruct:free";

    if (mode === "punctuation") {
      systemPrompt = "你是一個專業的中文標點符號處理助手。你的任務是接收用戶傳送的原始轉錄文字，並為其補上正確的繁體中文標點符號。規則：1. 禁止修改、刪除、增加任何原本的文字內容。2. 禁止修正語法。3. 禁止進行摘要。4. 必須 100% 保留所有原始字詞。只輸出加了標點的文字，不要有任何說明。";
    } else {
      systemPrompt = "請根據以下會議對話內容，整理成繁體中文重點摘要，包含主要討論事項與結論。使用條列式整理。";
    }

    console.log("使用模型:", targetModel);

    const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
        "Content-Type": "application/json",
        "HTTP-Referer": "https://meeting-app.local",
        "X-Title": "Meeting Transcription"
      },
      body: JSON.stringify({
        model: targetModel,
        messages: [
          { role: "system", content: systemPrompt },
          { role: "user", content: conversationText }
        ],
        temperature: 0.1,
      }),
    });

    const data = await response.json();
    
    console.log("OpenRouter 回應狀態:", response.status);
    console.log("OpenRouter 回應:", JSON.stringify(data).substring(0, 500));

    // 檢查 API 錯誤
    if (!response.ok || data.error) {
      const errorMsg = data.error?.message || data.error || `HTTP ${response.status}`;
      console.error("OpenRouter API 錯誤:", errorMsg);
      throw new Error(`OpenRouter 錯誤: ${errorMsg}`);
    }

    // 安全地取得結果
    if (!data.choices || !data.choices[0] || !data.choices[0].message) {
      console.error("OpenRouter 回應格式異常:", JSON.stringify(data));
      throw new Error("OpenRouter 回應格式異常，請檢查 API Key 和模型名稱");
    }

    const resultText = data.choices[0].message.content.trim();
    console.log("處理成功，結果長度:", resultText.length);

    return new Response(JSON.stringify({ 
      summary: resultText,
      text: resultText,
      success: true 
    }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });

  } catch (e) {
    console.error("Edge Function 錯誤:", e.message);
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
2. 前往 **Edge Functions** → 點擊進入 `rapid-action`
3. 點擊 **Code** 分頁
4. 用上方的程式碼**完全覆蓋** `index.ts`
5. 點擊 **Deploy**
6. 部署完成後，前往 **Logs** 分頁查看詳細錯誤訊息

---

## 🔍 常見錯誤排查

| 錯誤訊息 | 原因 | 解決方法 |
|---------|------|---------|
| `缺少 OPENROUTER_API_KEY` | 沒設定 Secret | 前往 Settings → Secrets 新增 |
| `401 Unauthorized` | API Key 無效 | 檢查 OpenRouter API Key 是否正確 |
| `404 Not Found` | 模型名稱錯誤 | 確認模型名稱正確 |
| `429 Too Many Requests` | 超過免費額度 | 等待或升級方案 |
