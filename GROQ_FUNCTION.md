# Groq Whisper v3 Turbo Edge Function

使用 Groq Cloud 的 Whisper v3 Turbo 模型進行高精確度的語音轉錄。

## 1. Supabase Secrets 設定
在 Supabase 專案中設定以下 Secret：
- `GROQ_API_KEY`: 您的 Groq API Key

## 2. 函數代碼 (`groq-speech`)

```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
};

serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  try {
    const supabaseUrl = Deno.env.get("SUPABASE_URL")!;
    const supabaseKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
    const supabase = createClient(supabaseUrl, supabaseKey);

    const authHeader = req.headers.get("authorization") || "";
    const token = authHeader.replace("Bearer ", "");
    
    const { data: { user }, error: authError } = await supabase.auth.getUser(token);
    if (authError || !user) {
      return new Response(JSON.stringify({ error: "Unauthorized" }), {
        status: 401, headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    const { audio, language, roomId } = await req.json();
    
    if (!audio || audio.length < 5000) {
      return new Response(JSON.stringify({ transcript: "", success: true }), {
        headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    // 1. 取得 API Key (優先找資料庫，次找 Secret)
    let apiKey = Deno.env.get("GROQ_API_KEY");
    let accountId = null;

    // 取得房間創建者的 ID 以獲取其 Key
    let ownerId = user.id;
    if (roomId && roomId !== 'main-room') {
      const { data: room } = await supabase.from('rooms').select('created_by').eq('room_id', roomId).single();
      if (room && room.created_by) ownerId = room.created_by;
    }

    const { data: accounts } = await supabase
      .from('speech_api_accounts')
      .select('id, api_key')
      .eq('user_id', ownerId)
      .eq('provider', 'groq')
      .eq('api_exhausted', false)
      .order('is_active', { ascending: false })
      .limit(1);

    if (accounts && accounts.length > 0) {
      apiKey = accounts[0].api_key;
      accountId = accounts[0].id;
    }

    if (!apiKey) {
      throw new Error("No Groq API key found in database or secrets.");
    }

    // 2. 調用 Groq API
    const binaryAudio = Uint8Array.from(atob(audio), c => c.charCodeAt(0));
    const audioBlob = new Blob([binaryAudio], { type: "audio/webm" });

    const formData = new FormData();
    formData.append("file", audioBlob, "audio.webm");
    formData.append("model", "whisper-large-v3-turbo");
    formData.append("language", language === "zh-TW" ? "zh" : "en");
    formData.append("prompt", "這是一場專業會議轉錄。請確保將聽到的所有數字、金額、日期精確地格式化為『阿拉伯數字』。若音訊中僅有背景雜訊、完全安靜或無法辨識的人聲，請直接回傳『空字串』，不要產生任何文字。");

    const response = await fetch("https://api.groq.com/openai/v1/audio/transcriptions", {
      method: "POST",
      headers: { "Authorization": `Bearer ${apiKey}` },
      body: formData,
    });

    const result = await response.json();

    if (!response.ok) {
        if ((response.status === 413 || response.status === 429) && accountId) {
             await supabase.from('speech_api_accounts').update({ api_exhausted: true }).eq('id', accountId);
        }
        throw new Error(result.error?.message || "Groq API 錯誤");
    }

    return new Response(JSON.stringify({ 
      transcript: (result.text || "").trim(), 
      success: true 
    }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" }
    });

  } catch (e) {
    return new Response(JSON.stringify({ error: String(e) }), {
      status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" }
    });
  }
});
```
