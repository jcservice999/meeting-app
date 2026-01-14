# OpenAI Whisper - 純 Prompt 版

**只用 Whisper prompt，不加 GPT**：

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
    const openaiKey = Deno.env.get("OPENAI_API_KEY");
    if (!openaiKey) {
      return new Response(JSON.stringify({ error: "No API key" }), {
        status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    const supabaseUrl = Deno.env.get("SUPABASE_URL")!;
    const supabaseKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
    const supabase = createClient(supabaseUrl, supabaseKey);

    const authHeader = req.headers.get("authorization") || "";
    const token = authHeader.replace("Bearer ", "");

    if (token) {
      const { error } = await supabase.auth.getUser(token);
      if (error) {
        return new Response(JSON.stringify({ error: "Auth failed" }), {
          status: 401, headers: { ...corsHeaders, "Content-Type": "application/json" }
        });
      }
    }

    const { audio, language } = await req.json();
    
    if (!audio || audio.length < 5000) {
      return new Response(JSON.stringify({ transcript: "", success: true }), {
        headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    const binaryAudio = Uint8Array.from(atob(audio), c => c.charCodeAt(0));
    const audioBlob = new Blob([binaryAudio], { type: "audio/webm" });

    const formData = new FormData();
    formData.append("file", audioBlob, "audio.webm");
    formData.append("model", "whisper-1");
    formData.append("language", language === "zh-TW" ? "zh" : language === "en-US" ? "en" : "zh");
    formData.append("response_format", "json");
    
    // 英文 prompt（較不會被回顯），設定為會議對話情境
    formData.append("prompt", "This is a meeting conversation recording. Transcribe only the actual spoken words. If there is silence, return empty.");

    const response = await fetch("https://api.openai.com/v1/audio/transcriptions", {
      method: "POST",
      headers: { "Authorization": `Bearer ${openaiKey}` },
      body: formData,
    });

    const result = await response.json();

    if (!response.ok) {
      return new Response(JSON.stringify({ error: result.error?.message }), {
        status: 400, headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    let transcript = (result.text || "").trim();
    
    // 只過濾太短的結果
    if (transcript.length < 2) {
      transcript = "";
    }

    console.log("結果:", transcript || "(空)");

    return new Response(JSON.stringify({ transcript, success: true }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" }
    });

  } catch (e) {
    console.error("錯誤:", e);
    return new Response(JSON.stringify({ error: String(e) }), {
      status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" }
    });
  }
});
```

**變更**：
- 使用**英文 prompt**（較不會被中文音訊回顯）
- 移除所有硬編碼過濾
- 只用 Whisper 自己的判斷

請更新後測試！如果還是有問題，可能真的需要考慮本地 Whisper 或其他方案。
