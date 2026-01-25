# AssemblyAI 語音轉錄 - Edge Function (穩定強化版)

## 完整程式碼

函數名稱：`assemblyai-speech`

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

    const { audio, language, roomId, wordBoost } = await req.json();
    
    // 🔴 修正：更寬鬆的音訊長度檢查，避免短語被過濾
    if (!audio || audio.length < 1000) {
      return new Response(JSON.stringify({ transcript: "", success: true }), {
        headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    let ownerId = user.id;
    if (roomId && roomId !== 'main-room') {
      const { data: room } = await supabase.from('rooms').select('created_by').eq('room_id', roomId).single();
      if (room && room.created_by) ownerId = room.created_by;
    }

    const { data: accounts } = await supabase
      .from('speech_api_accounts')
      .select('id, api_key')
      .eq('user_id', ownerId)
      .eq('provider', 'assemblyai')
      .eq('api_exhausted', false)
      .limit(1);

    if (!accounts || accounts.length === 0) {
        throw new Error("No active AssemblyAI key found");
    }

    const apiKey = accounts[0].api_key;
    const accountId = accounts[0].id;

    // 1. 上傳
    const binaryAudio = Uint8Array.from(atob(audio), c => c.charCodeAt(0));
    const uploadRes = await fetch("https://api.assemblyai.com/v2/upload", {
      method: "POST",
      headers: { "Authorization": apiKey },
      body: binaryAudio
    });

    if (!uploadRes.ok) throw new Error("Upload to AssemblyAI failed");

    const { upload_url } = await uploadRes.json();

    // 2. 轉錄
    const transcriptRes = await fetch("https://api.assemblyai.com/v2/transcript", {
      method: "POST",
      headers: { "Authorization": apiKey, "Content-Type": "application/json" },
      body: JSON.stringify({
        audio_url: upload_url,
        language_code: language === "zh-TW" ? "zh" : "en",
        speech_model: "best",
        word_boost: wordBoost || []
      })
    });

    if (!transcriptRes.ok) throw new Error("Transcription request failed");
    const { id: transcriptId } = await transcriptRes.json();

    // 3. 輪詢結果 (最多等待 25 秒，增加穩定性)
    let transcript = "";
    for (let i = 0; i < 25; i++) {
      await new Promise(r => setTimeout(r, 1000));
      const pollRes = await fetch(`https://api.assemblyai.com/v2/transcript/${transcriptId}`, {
        headers: { "Authorization": apiKey }
      });
      const poll = await pollRes.json();
      
      if (poll.status === "completed") {
        transcript = poll.text || "";
        break;
      } else if (poll.status === "error") {
        throw new Error(poll.error);
      }
    }

    return new Response(JSON.stringify({ transcript, success: true }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" }
    });

  } catch (e) {
    console.error("AssemblyAI Edge Error:", e.message);
    return new Response(JSON.stringify({ error: e.message, success: false }), {
      status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" }
    });
  }
});
```
