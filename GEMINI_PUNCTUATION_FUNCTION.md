# gemini-punctuation Edge Function (OpenRouter 完全免費版)

透過 **OpenRouter** 調用 **Gemini 2.0 Flash (Free版)** 模型，確保完全免費調用。

## 1. 費用說明
- **Groq**: 顯示的 $0.03 為「預估費用」，只要您不點擊升級 (Upgrade) 就不會扣款。
- **OpenRouter**: 本代碼已更換為 `:free` 模型 ID，確保不再扣除您的餘額。

## 2. 程式碼

請複製以下整個程式碼區塊，貼到您的 Supabase Edge Function 中。

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
    originalText = (body.text || "").trim();
    
    if (!originalText) {
      return new Response(JSON.stringify({ result: "", success: true }), {
        headers: { ...corsHeaders, "Content-Type": "application/json" },
      });
    }

    const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY");
    if (!OPENROUTER_API_KEY) {
      console.error("缺少 OPENROUTER_API_KEY");
      return new Response(JSON.stringify({ result: originalText, success: false, error: "Missing API Key" }), {
        headers: { ...corsHeaders, "Content-Type": "application/json" },
      });
    }

    // 🔴 關鍵更新：使用帶有 :free 標籤的模型以確保免費
    const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
        "Content-Type": "application/json",
        "HTTP-Referer": "https://supabase.local",
        "X-Title": "My Free Meeting App"
      },
      body: JSON.stringify({
        model: "google/gemini-2.0-flash-exp:free", 
        messages: [
          {
            role: "system",
            content: `你是一位專門處理語音轉錄稿的標點專家。
任務規則（必須死守）：
1. 【強制添加標點】：輸出結果的「每一句話末尾」必須有結束標點（。、？、！）。
2. 【標點補全】：在語氣助詞（啊、啦、喔、吧、嗎）後方強制加上標點。
3. 【禁止省略】：不要刪除任何文字，包括贅字或髒話。
4. 【同音字修正】：修正錯別字。

範例：
輸入：之前應該要把那個prompt留下來
輸出：之前應該要把那個 prompt 留下來。

輸入：欸你有聽說嗎那個人真的超級雞巴的啦我就說不用理他啊對不對
輸出：欸！你有聽說嗎？那個人真的超級雞巴的啦！我就說不用理他啊，對不對？

只回傳校對後的文字，不要解釋。`
          },
          {
            role: "user",
            content: originalText
          }
        ],
        temperature: 0.1 // 降低隨機性，確保標點穩定
      })
    });

    const data = await response.json();
    
    if (!response.ok) {
        console.error("OpenRouter 錯誤:", JSON.stringify(data));
        return new Response(JSON.stringify({ result: originalText, success: false, error: data.error?.message }), {
            headers: { ...corsHeaders, "Content-Type": "application/json" },
        });
    }

    let result = data.choices?.[0]?.message?.content?.trim() || originalText;

    // 後處理：若 AI 依然偷懶沒給結局標點，強制補個句號
    const punctuationRegex = /[。？！.?!]$/;
    if (result && !punctuationRegex.test(result)) {
        result += "。";
    }

    return new Response(JSON.stringify({ result, success: true }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });

  } catch (e) {
    console.error("Edge Function 錯誤:", e.message);
    return new Response(JSON.stringify({ result: originalText, success: false, error: e.message }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });
  }
});
```
