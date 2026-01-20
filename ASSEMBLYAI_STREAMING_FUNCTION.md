# AssemblyAI Streaming - Token Edge Function

## 函數名稱：`assemblyai-token`

在 Supabase Dashboard 建立這個 Edge Function：

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

    const { roomId } = await req.json();

    // 取得房間創建者的 ID
    let ownerId = user.id;
    
    if (roomId && roomId !== 'main-room') {
      const { data: room } = await supabase
        .from('rooms')
        .select('created_by')
        .eq('room_id', roomId)
        .single();
      
      if (room && room.created_by) {
        ownerId = room.created_by;
      }
    }

    // 取得 API Key
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
        error: "No active AssemblyAI API key" 
      }), {
        status: 400, headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    const apiKey = accounts[0].api_key;

    // 取得臨時 streaming token (使用穩定版 v2 Real-time API)
    const tokenRes = await fetch("https://api.assemblyai.com/v2/realtime/token", {
      method: "POST",
      headers: {
        "Authorization": apiKey,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({ expires_in: 3600 })
    });

    if (!tokenRes.ok) {
      if (tokenRes.status === 402 || tokenRes.status === 429) {
        await supabase.from('speech_api_accounts')
          .update({ api_exhausted: true, exhausted_at: new Date().toISOString() })
          .eq('id', accounts[0].id);
      }
      const errText = await tokenRes.text();
      throw new Error(`Failed to get token: ${errText}`);
    }

    const { token: streamToken } = await tokenRes.json();

    return new Response(JSON.stringify({ 
      token: streamToken,
      success: true 
    }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" }
    });

  } catch (e) {
    console.error("Error:", e);
    return new Response(JSON.stringify({ error: String(e) }), {
      status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" }
    });
  }
});
```

---

## 部署步驟

1. 在 Supabase Dashboard → Edge Functions → 建立 `assemblyai-token`
2. 貼上上面的程式碼
3. 關閉 JWT Verification
4. 部署
