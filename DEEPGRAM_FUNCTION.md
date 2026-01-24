# Deepgram 語音轉錄 - Edge Function

## 完整程式碼

函數名稱：`deepgram-speech`

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

    // 取得房間創建者的 ID
    let ownerId = user.id;
    let isVisitor = false;
    if (roomId && roomId !== 'main-room') {
      const { data: room } = await supabase
        .from('rooms')
        .select('created_by')
        .eq('room_id', roomId)
        .single();
      if (room && room.created_by) {
          ownerId = room.created_by;
          if (ownerId !== user.id) isVisitor = true;
      }
    }

    console.log(`Checking Deepgram key for ownerId: ${ownerId} (Current User: ${user.id})`);

    // 取得有效 Deepgram API Key (優先找 Room Creator)
    let { data: accounts } = await supabase
      .from('speech_api_accounts')
      .select('id, api_key, provider')
      .eq('user_id', ownerId)
      .eq('provider', 'deepgram')
      .eq('api_exhausted', false)
      .order('is_active', { ascending: false })
      .limit(1);

    // 如果 Creator 沒設定，且當前使用者不是 Creator，則嘗試用當前使用者的 Key
    if ((!accounts || accounts.length === 0) && isVisitor) {
        console.log(`Room creator has no Deepgram key. Falling back to current user: ${user.id}`);
        const { data: visitorAccounts } = await supabase
          .from('speech_api_accounts')
          .select('id, api_key, provider')
          .eq('user_id', user.id)
          .eq('provider', 'deepgram')
          .eq('api_exhausted', false)
          .order('is_active', { ascending: false })
          .limit(1);
        accounts = visitorAccounts;
    }

    if (!accounts || accounts.length === 0) {
      console.error(`No Deepgram API key found for owner ${ownerId} or visitor ${user.id}`);
      return new Response(JSON.stringify({ 
        error: "No active Deepgram API key found in this room. Please set up one in settings." 
      }), {
        status: 400, headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    const apiKey = accounts[0].api_key;
    const accountId = accounts[0].id;
    console.log(`Using Deepgram account: ${accountId} (Provider: ${accounts[0].provider})`);

    // 呼叫 Deepgram API
    // 優化數字辨識參數：numbers=true 與 dictation=true 對數字極其敏感
    const params = new URLSearchParams({
      model: "nova-2",
      language: language === "zh-TW" ? "zh-TW" : "en",
      punctuate: "true",
      numbers: "true",      // 強制數字化
      dictation: "true",    // 聽寫模式
      smart_format: "true", // 格式化小數點與貨幣
      filler_words: "true", // 保留贅詞有助於捕捉快速讀出的點與零
      no_delay: "true",
    });

    const url = `https://api.deepgram.com/v1/listen?${params.toString()}`;
    
    const binaryAudio = Uint8Array.from(atob(audio), c => c.charCodeAt(0));

    const response = await fetch(url, {
      method: "POST",
      headers: {
        "Authorization": `Token ${apiKey}`,
        "Content-Type": "audio/webm",
      },
      body: binaryAudio
    });

    if (!response.ok) {
        if (response.status === 401 || response.status === 402 || response.status === 429) {
             await supabase.from('speech_api_accounts')
              .update({ api_exhausted: true, exhausted_at: new Date().toISOString() })
              .eq('id', accountId);
        }
        const errText = await response.text();
        throw new Error(`Deepgram API error: ${errText}`);
    }

    const result = await response.json();
    const transcript = result.results?.channels[0]?.alternatives[0]?.transcript || "";

    return new Response(JSON.stringify({ transcript, success: true }), {
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

## 部署步驟

1. 在 Supabase Dashboard → Edge Functions → 建立 `deepgram-speech`
2. 貼上上面的程式碼
3. 確保關閉 JWT Verification
4. 部署
