# AssemblyAI Webhook 回調 - Edge Function

## 完整程式碼

函數名稱：`assemblyai-webhook`

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

    // AssemblyAI 發送的 webhook payload
    const { transcript_id, status } = await req.json();
    console.log("📨 收到 Webhook:", transcript_id, status);

    // 從 URL 參數取得 roomId 和 userId
    const url = new URL(req.url);
    const roomId = url.searchParams.get("roomId");
    const userId = url.searchParams.get("userId");

    if (!roomId || !userId) {
      console.error("缺少 roomId 或 userId");
      return new Response(JSON.stringify({ error: "Missing params" }), {
        status: 400, headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    if (status === "completed") {
      // 取得房間創建者的 API Key
      const { data: room } = await supabase
        .from('rooms')
        .select('created_by')
        .eq('room_id', roomId)
        .single();

      let ownerId = userId;
      if (room && room.created_by) {
        ownerId = room.created_by;
      }

      const { data: accounts } = await supabase
        .from('speech_api_accounts')
        .select('api_key')
        .eq('user_id', ownerId)
        .eq('provider', 'assemblyai')
        .eq('api_exhausted', false)
        .limit(1);

      if (!accounts || accounts.length === 0) {
        console.error("找不到 API Key");
        return new Response(JSON.stringify({ error: "No API key" }), {
          status: 400, headers: { ...corsHeaders, "Content-Type": "application/json" }
        });
      }

      const apiKey = accounts[0].api_key;

      // 取得完整轉錄結果
      const transcriptRes = await fetch(`https://api.assemblyai.com/v2/transcript/${transcript_id}`, {
        headers: { "Authorization": apiKey }
      });
      const transcript = await transcriptRes.json();

      if (transcript.text && transcript.text.trim()) {
        console.log("✅ 轉錄結果:", transcript.text.substring(0, 50) + "...");

        // 取得使用者名稱
        const { data: userData } = await supabase.auth.admin.getUserById(userId);
        const userName = userData?.user?.user_metadata?.full_name || 
                        userData?.user?.email?.split('@')[0] || 
                        '未知使用者';

        // 儲存到 captions 表
        await supabase.from('captions').insert({
          meeting_id: roomId,
          user_id: userId,
          user_name: userName,
          content: transcript.text
        });

        console.log("💾 字幕已儲存");
      }
    } else if (status === "error") {
      console.error("❌ 轉錄失敗:", transcript_id);
    }

    return new Response(JSON.stringify({ success: true }), {
      headers: { ...corsHeaders, "Content-Type": "application/json" }
    });

  } catch (e) {
    console.error("Webhook 處理失敗:", e);
    return new Response(JSON.stringify({ error: String(e) }), {
      status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" }
    });
  }
});
```

---

## 部署步驟

1. 在 Supabase Dashboard → Edge Functions → 建立新函數 `assemblyai-webhook`
2. 貼上上面的程式碼
3. **關閉 JWT Verification**（AssemblyAI 不會發送 JWT）
4. 部署

---

## Webhook URL 格式

```
https://inzqsdelrwxxlbcumaiw.supabase.co/functions/v1/assemblyai-webhook?roomId={roomId}&userId={userId}
```
