# AssemblyAI 語音轉錄 - Edge Function

## 完整程式碼（使用房間創建者的 API Key）

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
    
    if (!audio || audio.length < 5000) {
      return new Response(JSON.stringify({ transcript: "", success: true }), {
        headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    // 取得房間創建者的 ID
    let ownerId = user.id; // 預設使用當前使用者
    
    if (roomId && roomId !== 'main-room') {
      const { data: room } = await supabase
        .from('rooms')
        .select('created_by')
        .eq('room_id', roomId)
        .single();
      
      if (room && room.created_by) {
        ownerId = room.created_by;
        console.log("Using room creator's API key:", ownerId);
      }
    }

    // 取得房間創建者的有效 API Key
    const { data: accounts } = await supabase
      .from('speech_api_accounts')
      .select('id, api_key')
      .eq('user_id', ownerId)
      .eq('provider', 'assemblyai')
      .eq('api_exhausted', false)
      .order('is_active', { ascending: false })
      .limit(1);

    if (!accounts || accounts.length === 0) {
      return new Response(JSON.stringify({ 
        error: "Room creator has no active AssemblyAI API key" 
      }), {
        status: 400, headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    const apiKey = accounts[0].api_key;
    const accountId = accounts[0].id;

    // 1. 上傳音訊到 AssemblyAI
    const binaryAudio = Uint8Array.from(atob(audio), c => c.charCodeAt(0));
    
    const uploadRes = await fetch("https://api.assemblyai.com/v2/upload", {
      method: "POST",
      headers: { "Authorization": apiKey },
      body: binaryAudio
    });

    if (!uploadRes.ok) {
      if (uploadRes.status === 402 || uploadRes.status === 429) {
        await supabase.from('speech_api_accounts')
          .update({ api_exhausted: true, exhausted_at: new Date().toISOString() })
          .eq('id', accountId);
        return new Response(JSON.stringify({ error: "API quota exhausted" }), {
          status: 400, headers: { ...corsHeaders, "Content-Type": "application/json" }
        });
      }
      throw new Error("Upload failed");
    }

    const { upload_url } = await uploadRes.json();

    // 2. 建立轉錄（使用 Webhook 取代輪詢）
    const webhookUrl = `${Deno.env.get("SUPABASE_URL")}/functions/v1/assemblyai-webhook?roomId=${roomId}&userId=${user.id}`;
    
    const transcriptRes = await fetch("https://api.assemblyai.com/v2/transcript", {
      method: "POST",
      headers: { "Authorization": apiKey, "Content-Type": "application/json" },
      body: JSON.stringify({
        audio_url: upload_url,
        language_code: language === "zh-TW" ? "zh" : "en",
        word_boost: wordBoost || [],
        boost_param: "high",
        punctuate: true,
        format_text: true,
        webhook_url: webhookUrl
      })
    });

    const { id: transcriptId } = await transcriptRes.json();
    console.log("📤 已提交轉錄，ID:", transcriptId);

    // 立即返回，結果會透過 webhook 送達
    return new Response(JSON.stringify({ 
      submitted: true, 
      transcriptId,
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

---

## 更新步驟

1. 在 Supabase Dashboard → Edge Functions → `assemblyai-speech`
2. 用上面的程式碼替換
3. 確保關閉 JWT Verification
4. 部署
